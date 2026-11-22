# Backend Refactoring Specification: Multi-Stage Competition System

## 1. Objective & Cleanup Scope
Upgrade the existing Python + FastAPI + PostgreSQL single-submission CRUD system into a multi-stage competition backend (Preliminary / Semifinal / Final) with **minimal invasive changes**[span_0](start_span)[span_0](end_span).
- **Cleanup First:** Remove any previously generated "stage registration" (阶段报名) routes, schemas, or redundant tables. No separate stage registration endpoint is needed[span_1](start_span)[span_1](end_span).

---

## 2. Strict Guardrails (MUST OBEY)
1. **Preserve Existing User Routes:** Keep the exact URL paths and HTTP methods for the 5 user endpoints[span_2](start_span)[span_2](end_span):
   - `GET /submissions` (fetch current user's all active submissions)
   - `GET /submission/{id}` (fetch single submission detail)
   - `POST /submissions` (create submission — acts as registration + preliminary submit)[span_3](start_span)[span_3](end_span)
   - `PUT /submission/{id}` (update submission in any open stage)[span_4](start_span)[span_4](end_span)
   - `DELETE /submission/{id}` (soft delete submission)
2. **Reuse Existing Modules:** Directly reuse existing authentication (`get_current_user`), MinIO upload utilities, AI evaluation caller, and the **existing team member / NT account verification service/function**. Do NOT rewrite them.
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
- `prelim_config`, `semi_config`, `final_config`: `JSONB`, nullable=False, default=`dict`[span_5](start_span)[span_5](end_span)
  - Each stores round-specific `file_requirements` (allowing different file formats per round, e.g., 1 PPT + 1 PDF in Prelim) and `ai_prompt`[span_6](start_span)[span_6](end_span):
    ```json
    {
      "ai_prompt": "Stage-specific AI evaluation prompt...",
      "file_requirements": [
        { "key": "prelim_ppt", "label": "初赛展示PPT", "required": true, "allowed_extensions": [".ppt", ".pptx"] },
        { "key": "prelim_pdf", "label": "初赛项目计划书", "required": true, "allowed_extensions": [".pdf"] }
      ]
    }
    ```

### Table 2: `submissions`
Keep existing core columns and ensure the following fields exist:
- `id`: `Integer`, PK
- `user_id`: `Integer`, FK(`users.id`), indexed
- `factory_zone`: `String(64)`, nullable=False, indexed (厂区)
- `topic`: `String(32)`, nullable=False, indexed (主题，三选一枚举)
- `title`: `String(255)`, nullable=False (投稿名称)
- `description`: `Text`, nullable=True
- `members`: `JSONB`, nullable=False, default=`list`
  - Stores verified member objects (including the submitter):
    `[{"nt_account": "...", "name": "...", "factory_zone": "...", "employee_id": "...", "verified": true}]`
- `files`: `JSONB`, nullable=False, default=`dict`
  - Stores MinIO file metadata keyed by file slot:
    `{"prelim_ppt": {"filename": "...", "minio_key": "...", "url": "..."}}`
- `status`: `String(32)`, nullable=False, default=`"REGISTERED"`, indexed[span_7](start_span)[span_7](end_span)
  - Enum: `REGISTERED` (报名), `ADVANCED_SEMI` (晋级复赛), `ADVANCED_FINAL` (晋级决赛)[span_8](start_span)[span_8](end_span)
- `ai_messages`: `JSONB`, nullable=False, default=`dict`[span_9](start_span)[span_9](end_span)
  - Stores AI feedback isolated by round key: `{"PRELIM": "...", "SEMI": "...", "FINAL": "..."}`[span_10](start_span)[span_10](end_span)
- `created_at`, `updated_at`: `DateTime(timezone=True)`
- `deleted_at`: `DateTime(timezone=True)`, nullable=True (soft delete)

---

## 4. Stage State Machine & Permission Matrix

### A. Real-Time Stage Resolution (`get_current_stage`)
Check `now = datetime.now(timezone.utc)` in reverse chronological order:
1. `final_end_at` and `now >= final_end_at` -> `FINAL_ENDED`[span_11](start_span)[span_11](end_span)
2. `final_start_at` and `now >= final_start_at` -> `FINAL_OPEN`[span_12](start_span)[span_12](end_span)
3. `semi_end_at` and `now >= semi_end_at` -> `SEMI_ENDED`[span_13](start_span)[span_13](end_span)
4. `semi_start_at` and `now >= semi_start_at` -> `SEMI_OPEN`[span_14](start_span)[span_14](end_span)
5. `prelim_end_at` and `now >= prelim_end_at` -> `PRELIM_ENDED`[span_15](start_span)[span_15](end_span)
6. `prelim_start_at` and `now >= prelim_start_at` -> `PRELIM_OPEN`[span_16](start_span)[span_16](end_span)
7. Otherwise -> `ALL_CLOSED`[span_17](start_span)[span_17](end_span)

### B. Access & AI Feedback Rules
| `global_stage` | Active Round | Writable | Required `status` | Returned `ai_message`[span_18](start_span)[span_18](end_span) |
| :--- | :--- | :--- | :--- | :--- |
| `ALL_CLOSED` | `None` | `False` (403) | `None` | `None` |
| `PRELIM_OPEN` | `"PRELIM"` | `True` (`POST`/`PUT`/`DELETE`) | `"REGISTERED"` | `ai_messages.get("PRELIM")` |
| `PRELIM_ENDED` | `"PRELIM"` | `False` (403) | `None` | `ai_messages.get("PRELIM")` |
| `SEMI_OPEN` | `"SEMI"` | `True` (`PUT`/`DELETE` only) | `"ADVANCED_SEMI"` | `ai_messages.get("SEMI")` |
| `SEMI_ENDED` | `"SEMI"` | `False` (403) | `None` | `ai_messages.get("SEMI")` |
| `FINAL_OPEN` | `"FINAL"` | `True` (`PUT`/`DELETE` only) | `"ADVANCED_FINAL"` | `ai_messages.get("FINAL")` |
| `FINAL_ENDED` | `"FINAL"` | `False` (403) | `None` | `ai_messages.get("FINAL")` |

---

## 5. Endpoint Behavior Rules
1. **Member Verification (`POST` & `PUT`):** Accept `member_nt_accounts` list from request (ensure `current_user.nt_account` is included and deduplicated). Call the **existing member verification service** to populate `members`. Reject with `400` if any account fails verification.
2. **File Merging & Validation (`POST` & `PUT`):**
   - On `PUT`, merge incoming `files` with existing `sub.files` (`merged = {**(sub.files or {}), **incoming}`) so users can update previous rounds' files while adding current round files[span_19](start_span)[span_19](end_span).
   - Validate that all `required: true` keys in the **active round's config** exist in `merged`, and every file extension in `merged` matches its slot's `allowed_extensions` across all stage configs.
3. **AI Feedback Isolation (`GET`, `POST`, `PUT`):**
   - On `POST` / `PUT`, invoke the existing AI function with the **active round's `ai_prompt`** and save result ONLY to `sub.ai_messages[active_round]`[span_20](start_span)[span_20](end_span).
   - On all responses, return full `files` dict, helper flags (`global_stage`, `can_edit`, `can_delete`, `current_file_requirements`), and **ONLY the active round's `ai_message`** (singular string or `null`, never historical rounds)[span_21](start_span)[span_21](end_span).
   - On `GET /submissions`, also include top-level `can_create` (`global_stage == "PRELIM_OPEN"`) and `templates` (the 4 bilingual files from `competition_configs`).
4. **Admin Endpoints:** Ensure Admin can list/filter all submissions (`factory_zone`, `topic`, `status`), batch-update `status` to promote submissions (`ADVANCED_SEMI` / `ADVANCED_FINAL`)[span_22](start_span)[span_22](end_span), and update `competition_configs` (timestamps, stage configs, and the 4 `templates` slots).

---

## 6. Execution Steps
1. Read existing codebase (models, schemas, user routers, admin routers, existing member verification function, existing AI caller).
2. Output a concise file-by-file change plan (including cleanup of any unwanted stage-registration endpoints) and wait for confirmation.
3. Implement changes cleanly after confirmation.
