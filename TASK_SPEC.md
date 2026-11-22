# Backend Refactoring Specification: Multi-Stage Competition System

## 1. Objective & Cleanup Scope
Upgrade the existing Python + FastAPI + PostgreSQL single-submission CRUD system into a multi-stage competition backend (Preliminary / Semifinal / Final) with **minimal invasive changes**[span_0](start_span)[span_0](end_span).
- **Cleanup First:** Remove any previously generated "stage registration" (阶段报名) routes, schemas, or redundant tables. No separate stage registration endpoint is needed[span_1](start_span)[span_1](end_span).

---

## 2. Strict Guardrails (MUST OBEY)
1. **Preserve Existing Routes:** Keep existing URL paths and HTTP methods for `POST /upload`, `GET /submissions`, `GET /submission/{id}`, `POST /submissions`, `PUT /submission/{id}`, and `DELETE /submission/{id}`[span_2](start_span)[span_2](end_span).
2. **Reuse Existing Modules:** Directly reuse existing authentication (`get_current_user`), MinIO client/upload helper, AI evaluation caller, and the **existing team member / NT account verification service/function**.
3. **No Background Schedulers:** Do NOT introduce Celery, APScheduler, or cron jobs. Resolve the current competition stage dynamically on every request by comparing `datetime.now(timezone.utc)` against configured timestamps.
4. **SQLAlchemy JSONB Mutation Rule:** Never mutate JSONB dicts/lists in-place (e.g., `sub.files.update(...)`). Always assign a brand-new copy (`sub.files = {**(sub.files or {}), **new_files}`) so SQLAlchemy detects changes and executes `UPDATE`.

---

## 3. Database Schema (PostgreSQL + SQLAlchemy)

### Table 1: `competition_configs` (Singleton `id=1`)
- `id`: `Integer`, PK, default `1`
- **Multi-Stage Timeline (UTC `DateTime(timezone=True)`, nullable):**
  - `prelim_start_at`, `prelim_end_at`
  - `semi_start_at`, `semi_end_at`
  - `final_start_at`, `final_end_at`
- `templates`: `JSONB`, nullable=False, default=`dict`
  - Used ONLY for Preliminary registration bilingual guides & templates (fixed 4 keys):
    ```json
    {
      "guide_zh":    { "filename": "...", "minio_key": "...", "url": "..." },
      "template_zh": { "filename": "...", "minio_key": "...", "url": "..." },
      "guide_en":    { "filename": "...", "minio_key": "...", "url": "..." },
      "template_en": { "filename": "...", "minio_key": "...", "url": "..." }
    }
    ```
- `prelim_config`, `semi_config`, `final_config`: `JSONB`, nullable=False, default=`dict`[span_3](start_span)[span_3](end_span)
  - **Preset File Requirements per Stage:**
    - **Prelim (`prelim_config`):** 1 PDF + 1 PPT
      ```json
      {
        "ai_prompt": "Preliminary round AI prompt...",
        "file_requirements": [
          { "key": "prelim_pdf", "label": "初赛项目计划书(PDF)", "required": true, "allowed_extensions": [".pdf"] },
          { "key": "prelim_ppt", "label": "初赛展示PPT", "required": true, "allowed_extensions": [".ppt", ".pptx"] }
        ]
      }
      ```
    - **Semifinal (`semi_config`):** 1 CAD file
      ```json
      {
        "ai_prompt": "Semifinal round AI prompt...",
        "file_requirements": [
          { "key": "semi_cad", "label": "复赛CAD图纸", "required": true, "allowed_extensions": [".dwg", ".dxf", ".step", ".stp", ".igs", ".iges"] }
        ]
      }
      ```
    - **Final (`final_config`):** 1 MP4 video
      ```json
      {
        "ai_prompt": "Final round AI prompt...",
        "file_requirements": [
          { "key": "final_video", "label": "决赛演示视频(MP4)", "required": true, "allowed_extensions": [".mp4"] }
        ]
      }
      ```

### Table 2: `submissions`
Keep existing core columns and ensure the following fields exist:
- `id`: `Integer`, PK
- `user_id`: `Integer`, FK(`users.id`), indexed
- `is_draft`: `Boolean`, nullable=False, default=`True`, indexed (becomes `False` once `POST /submissions` completes initial submission)
- `factory_zone`: `String(64)`, nullable=True (required when finalizing `POST /submissions`), indexed
- `topic`: `String(32)`, nullable=True (required when finalizing `POST /submissions`, 3-option enum), indexed
- `title`: `String(255)`, nullable=True (required when finalizing `POST /submissions`)
- `description`: `Text`, nullable=True
- `members`: `JSONB`, nullable=False, default=`list`
  - Verified member objects (including submitter): `[{"nt_account": "...", "name": "...", "factory_zone": "...", "employee_id": "...", "verified": true}]`
- `files`: `JSONB`, nullable=False, default=`dict`
  - MinIO metadata keyed by file slot: `{"prelim_pdf": {"filename": "...", "minio_key": "...", "url": "..."}, "prelim_ppt": {...}, "semi_cad": {...}, "final_video": {...}}`
- `status`: `String(32)`, nullable=False, default=`"REGISTERED"`, indexed[span_4](start_span)[span_4](end_span)
  - Enum: `REGISTERED` (报名), `ADVANCED_SEMI` (晋级复赛), `ADVANCED_FINAL` (晋级决赛)[span_5](start_span)[span_5](end_span)
- `ai_messages`: `JSONB`, nullable=False, default=`dict`[span_6](start_span)[span_6](end_span)
  - Isolated by round: `{"PRELIM": "...", "SEMI": "...", "FINAL": "..."}`[span_7](start_span)[span_7](end_span)
- `created_at`, `updated_at`: `DateTime(timezone=True)`
- `deleted_at`: `DateTime(timezone=True)`, nullable=True (soft delete)

---

## 4. Stage State Machine & Permission Matrix

### A. Real-Time Stage Resolution (`get_current_stage`)
Check `now = datetime.now(timezone.utc)` in reverse chronological order:
1. `final_end_at` and `now >= final_end_at` -> `FINAL_ENDED`[span_8](start_span)[span_8](end_span)
2. `final_start_at` and `now >= final_start_at` -> `FINAL_OPEN`[span_9](start_span)[span_9](end_span)
3. `semi_end_at` and `now >= semi_end_at` -> `SEMI_ENDED`[span_10](start_span)[span_10](end_span)
4. `semi_start_at` and `now >= semi_start_at` -> `SEMI_OPEN`[span_11](start_span)[span_11](end_span)
5. `prelim_end_at` and `now >= prelim_end_at` -> `PRELIM_ENDED`[span_12](start_span)[span_12](end_span)
6. `prelim_start_at` and `now >= prelim_start_at` -> `PRELIM_OPEN`[span_13](start_span)[span_13](end_span)
7. Otherwise -> `ALL_CLOSED`[span_14](start_span)[span_14](end_span)

### B. Access & AI Feedback Rules
| `global_stage` | Active Round | Writable | Required `status` | Returned `ai_message`[span_15](start_span)[span_15](end_span) |
| :--- | :--- | :--- | :--- | :--- |
| `ALL_CLOSED` | `None` | `False` (403) | `None` | `None` |
| `PRELIM_OPEN` | `"PRELIM"` | `True` (`POST`/`PUT`/`DELETE`) | `"REGISTERED"` | `ai_messages.get("PRELIM")` |
| `PRELIM_ENDED` | `"PRELIM"` | `False` (403) | `None` | `ai_messages.get("PRELIM")` |
| `SEMI_OPEN` | `"SEMI"` | `True` (`PUT`/`DELETE` only) | `"ADVANCED_SEMI"` | `ai_messages.get("SEMI")` |
| `SEMI_ENDED` | `"SEMI"` | `False` (403) | `None` | `ai_messages.get("SEMI")` |
| `FINAL_OPEN` | `"FINAL"` | `True` (`PUT`/`DELETE` only) | `"ADVANCED_FINAL"` | `ai_messages.get("FINAL")` |
| `FINAL_ENDED` | `"FINAL"` | `False` (403) | `None` | `ai_messages.get("FINAL")` |

---

## 5. Endpoint Specifications (Including Multi-File `/upload` Flow)

1. **`POST /upload` (Upload File to MinIO & Bind to `submission_id`):**
   - **Inputs:** `file` (multipart), `file_key` (e.g., `prelim_pdf`, `prelim_ppt`, `semi_cad`, `final_video`), `submission_id` (`Optional[int] = None`).
   - **Validation:** Look up `file_key` across stage configs (`prelim_config`, `semi_config`, `final_config`) and verify file extension against `allowed_extensions` BEFORE uploading to MinIO.
   - **Case A (`submission_id` is `None` — first file upload in Prelim):**
     - Guard: Require `global_stage == "PRELIM_OPEN"` (403 otherwise).
     - Create a draft `Submission` (`user_id=current_user.id`, `is_draft=True`, `status="REGISTERED"`).
     - Upload to MinIO, save `sub.files = {file_key: file_meta}`, commit, and return `{"submission_id": sub.id, "file_key": file_key, "file": file_meta}`.
   - **Case B (`submission_id` is provided — 2nd file in Prelim, or re-upload, or Semi CAD / Final MP4):**
     - Fetch `Submission` by `submission_id` (`user_id == current_user.id`, `deleted_at IS NULL`).
     - Guard: Require `global_stage` is `*_OPEN` and `sub.status == required_status` (403 otherwise).
     - Upload to MinIO, merge `sub.files = {**(sub.files or {}), file_key: file_meta}`, commit, and return `{"submission_id": sub.id, "file_key": file_key, "file": file_meta}`.

2. **`POST /submissions` (Finalize Preliminary Submission / Registration):**
   - **Guard:** Require `global_stage == "PRELIM_OPEN"`.
   - **Input:** `submission_id` (created during `/upload`), `factory_zone`, `topic`, `title`, `description`, `member_nt_accounts` (list of NT accounts), plus optional `files` dict.
   - **Logic:**
     - Fetch draft `Submission` (`id == submission_id`, `user_id == current_user.id`, `deleted_at IS NULL`). (If `submission_id` is omitted and `files` are passed directly, allow creating directly as fallback).
     - Merge any `files` in payload with `sub.files`, then validate all `required: true` files for `"PRELIM"` (`prelim_pdf` AND `prelim_ppt`) are present.
     - Ensure `current_user.nt_account` is in `member_nt_accounts` (deduplicated), call the **existing NT account verification service** to populate `sub.members` (reject with `400` if any account fails verification).
     - Set `sub.is_draft = False`, run existing AI evaluation using `prelim_config.ai_prompt`, store result in `sub.ai_messages = {"PRELIM": ai_msg}`, commit, and return formatted response.

3. **`PUT /submission/{id}` (Update Submission in Open Stages):**
   - **Guard:** Require `global_stage` is `*_OPEN`, `sub.user_id == current_user.id`, `is_draft == False`, `deleted_at IS NULL`, and `sub.status == required_status`.
   - **Logic:**
     - Merge incoming `files` (if any) with existing `sub.files` (files already uploaded via `POST /upload` with `submission_id` are already in `sub.files`).
     - Validate all `required: true` files for the **active round** exist in `sub.files` (e.g., `semi_cad` in Semi, `final_video` in Final) and all file extensions in `sub.files` match their slot rules (allows updating previous rounds' files)[span_16](start_span)[span_16](end_span).
     - If `member_nt_accounts` is provided, re-verify via existing NT verification service and update `sub.members`.
     - Run existing AI evaluation using **active round's `ai_prompt`**, update ONLY `sub.ai_messages[active_round]`, commit, and return formatted response[span_17](start_span)[span_17](end_span).

4. **`GET /submissions` & `GET /submission/{id}`:**
   - Filter out `deleted_at IS NOT NULL` AND `is_draft == True` (so incomplete upload drafts are hidden).
   - Return full `files` dict, `members`, helper flags (`global_stage`, `can_edit`, `can_delete`, `current_file_requirements`), and **ONLY the active round's `ai_message`**[span_18](start_span)[span_18](end_span).
   - `GET /submissions` also includes top-level `can_create` (`global_stage == "PRELIM_OPEN"`) and `templates` (the 4 bilingual files from `competition_configs`).

5. **`DELETE /submission/{id}`:**
   - Soft delete (`sub.deleted_at = datetime.now(timezone.utc)`) under the same stage & `status` guards as `PUT`.

---

## 6. Execution Steps
1. Inspect existing `/upload` endpoint, `Submission` model, `CompetitionConfig` (or existing config model), user/admin routers, existing NT account verification function, and AI evaluation caller.
2. Output a concise file-by-file change & cleanup plan and pause for confirmation.
3. Implement after confirmation.
