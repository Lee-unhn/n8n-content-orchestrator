<!--
============================================================
ComfyUI integration patch — n8n Resolve Image node
============================================================

替換掉 v4 workflow 內 Resolve Image node 的 jsCode 的「Step 2 Pollinations」段，改成 call 本機 ComfyUI HTTP API。

ComfyUI server 要先 start 起來，跑在 host port 8188。從 n8n container 內用 `http://host.docker.internal:8188` 連到 host。

============================================================
-->

## 整個 Resolve Image node 替換版本（用這個整段貼上）

雙擊 v4 內 Resolve Image node → 把 JS 框全選刪除 → 貼這段：

```javascript
// Resolve image: Drive folder scan → ComfyUI local SDXL anime mascot fallback
const meta = $input.first().json.metadata;
const upstream = $input.first().json;
const topic = meta.image_topic_for_gen || meta.topic;

let image_base64 = null;
let image_source = 'none';
let image_error = null;
let drive_file_used = null;

const wfData = $getWorkflowStaticData('global');
wfData.processed_drive_ids = wfData.processed_drive_ids || [];

const COMFY_BASE = $env.COMFYUI_BASE_URL || 'http://host.docker.internal:8188';
const COMFY_CKPT = $env.COMFYUI_CHECKPOINT || 'animagineXL40_v40.safetensors';

// ComfyUI workflow template (API format, txt2img SDXL anime mascot 1024x1024)
function buildComfyWorkflow(positivePrompt, seed) {
  return {
    "3": {
      "inputs": {
        "seed": seed,
        "steps": 28,
        "cfg": 6.5,
        "sampler_name": "euler_ancestral",
        "scheduler": "normal",
        "denoise": 1,
        "model": ["4", 0],
        "positive": ["6", 0],
        "negative": ["7", 0],
        "latent_image": ["5", 0]
      },
      "class_type": "KSampler"
    },
    "4": {
      "inputs": { "ckpt_name": COMFY_CKPT },
      "class_type": "CheckpointLoaderSimple"
    },
    "5": {
      "inputs": { "width": 1024, "height": 1024, "batch_size": 1 },
      "class_type": "EmptyLatentImage"
    },
    "6": {
      "inputs": {
        "text": "masterpiece, best quality, very aesthetic, absurdres, 1girl, solo, kemonomimi, cat ears, headphones around neck, long brown hair, side ponytail, choker necklace, dark blue shirt, soft cinematic lighting, depth of field, looking at viewer, upper body, professional illustration, high detail, " + positivePrompt,
        "clip": ["4", 1]
      },
      "class_type": "CLIPTextEncode"
    },
    "7": {
      "inputs": {
        "text": "lowres, bad anatomy, bad hands, text, error, missing fingers, extra digit, fewer digits, cropped, worst quality, low quality, normal quality, jpeg artifacts, signature, watermark, username, blurry, artist name, ugly, deformed, disfigured, mutated, multiple views, multiple girls, nsfw",
        "clip": ["4", 1]
      },
      "class_type": "CLIPTextEncode"
    },
    "8": {
      "inputs": { "samples": ["3", 0], "vae": ["4", 2] },
      "class_type": "VAEDecode"
    },
    "9": {
      "inputs": { "filename_prefix": "n8n_orchestrator", "images": ["8", 0] },
      "class_type": "SaveImage"
    }
  };
}

try {
  // ---- Step 1: Drive folder scan (priority) ----
  if ($env.DRIVE_API_KEY && $env.DRIVE_INBOX_FOLDER_ID) {
    const listResp = await this.helpers.httpRequest({
      method: 'GET',
      url: 'https://www.googleapis.com/drive/v3/files',
      qs: {
        q: `'${$env.DRIVE_INBOX_FOLDER_ID}' in parents and trashed=false and mimeType contains 'image/'`,
        key: $env.DRIVE_API_KEY,
        fields: 'files(id,name,mimeType,createdTime)',
        orderBy: 'createdTime'
      },
      json: true
    });
    const files = (listResp.files || []).filter(f => f.mimeType && f.mimeType.startsWith('image/'));
    if (files.length > 0) {
      const fresh = files.find(f => !wfData.processed_drive_ids.includes(f.id));
      if (fresh) {
        const buf = await this.helpers.httpRequest({
          method: 'GET',
          url: `https://www.googleapis.com/drive/v3/files/${fresh.id}?alt=media&key=${$env.DRIVE_API_KEY}`,
          encoding: 'arraybuffer',
          json: false,
          returnFullResponse: false
        });
        image_base64 = Buffer.from(buf).toString('base64');
        image_source = `drive:${fresh.id}`;
        drive_file_used = { id: fresh.id, name: fresh.name };
        wfData.processed_drive_ids.push(fresh.id);
      }
    }
  }

  // ---- Step 2: ComfyUI local SDXL anime generation ----
  if (!image_base64 && topic) {
    let positivePrompt = typeof topic === 'string' ? topic : String(topic);
    // 中文 topic 直接送 (SDXL CLIP 接受中文較弱，但仍能 work)
    // 想翻成英文可手動 translate 或日後加 LLM translate

    const seed = Math.floor(Math.random() * 1e15);
    const workflow = buildComfyWorkflow(positivePrompt, seed);
    const clientId = 'n8n-orchestrator';

    // 1. POST /prompt — 提交 workflow
    const submitResp = await this.helpers.httpRequest({
      method: 'POST',
      url: `${COMFY_BASE}/prompt`,
      body: { prompt: workflow, client_id: clientId },
      json: true,
      timeout: 30000
    });
    const promptId = submitResp.prompt_id;
    if (!promptId) throw new Error('ComfyUI no prompt_id: ' + JSON.stringify(submitResp).slice(0, 300));

    // 2. Poll /history — 等完成 (SDXL 28 steps on 3060 ≈ 25-40s)
    let history = null;
    const maxPollSec = 120;
    for (let i = 0; i < maxPollSec / 2; i++) {
      await new Promise(r => setTimeout(r, 2000));
      const hr = await this.helpers.httpRequest({
        method: 'GET',
        url: `${COMFY_BASE}/history/${promptId}`,
        json: true,
        timeout: 10000
      });
      if (hr[promptId] && hr[promptId].outputs && Object.keys(hr[promptId].outputs).length > 0) {
        history = hr[promptId];
        break;
      }
    }
    if (!history) throw new Error('ComfyUI poll timeout (>120s)');

    // 3. 找 output image filename
    let imgFile = null;
    for (const nodeId of Object.keys(history.outputs)) {
      const out = history.outputs[nodeId];
      if (out.images && out.images.length > 0) {
        imgFile = out.images[0];
        break;
      }
    }
    if (!imgFile) throw new Error('ComfyUI history no images');

    // 4. GET /view — 下載 image bytes
    const buf = await this.helpers.httpRequest({
      method: 'GET',
      url: `${COMFY_BASE}/view`,
      qs: { filename: imgFile.filename, subfolder: imgFile.subfolder || '', type: 'output' },
      encoding: 'arraybuffer',
      json: false,
      returnFullResponse: false,
      timeout: 30000
    });
    image_base64 = Buffer.from(buf).toString('base64');
    image_source = 'comfyui:sdxl-anime';
    image_error = `prompt_seed: ${seed}, prompt: ${positivePrompt.slice(0, 100)}`;
  }

  if (!image_base64) {
    image_error = image_error || 'No new Drive image and ComfyUI not invoked';
  }
} catch (e) {
  image_error = `comfyui: ${e.message}`;
}

return [{
  json: {
    ...upstream,
    image_base64,
    image_source,
    image_error,
    has_image: !!image_base64,
    drive_file_used,
    processed_count: wfData.processed_drive_ids.length
  }
}];
```

---

## 同步要做的事

1. **.env 新增 2 個變數**：

```bash
# ComfyUI (local AI image gen)
COMFYUI_BASE_URL=http://host.docker.internal:8188
COMFYUI_CHECKPOINT=animagineXL40_v40.safetensors
```

2. **n8n docker-compose 環境變數**：確認 n8n container 有 `host.docker.internal` resolve（Docker Desktop Windows 預設有）

3. **AI 標註偵測規則改**：之前 v4 內 FB/IG/Threads node 偵測 `image_source.startsWith('pollinations.ai')`，現在改成 `image_source.startsWith('comfyui')` 或 `image_source.startsWith('pollinations.ai')`（兩個都當 AI 生圖）

```javascript
// 三個 publish node 內這行
const aiNote = (imageSource.startsWith('comfyui') || imageSource.startsWith('pollinations.ai')) ? '\n\n（圖片 AI 生成）' : '';
```

---

## 啟動流程

1. ComfyUI server 跑起來（`start_comfyui_server.bat`）
2. 確認 http://localhost:8188 在瀏覽器能開 ComfyUI UI
3. 確認 `models/checkpoints/animagineXL40_v40.safetensors` 在
4. 從 n8n container 內 test connectivity：`docker exec n8n-content-orchestrator wget -O- http://host.docker.internal:8188/system_stats`
5. e2e test 跑 v4 → 看 Final Log 內 `image_source: 'comfyui:sdxl-anime'`
