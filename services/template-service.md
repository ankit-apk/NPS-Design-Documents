# 📩 Template Service – LLD

_“Centralized source of truth for user communication templates”_

---

## 🧱 High-Level Overview

### 🔗 Relationship

```text
Template Service  <--->  Notification Service
       ↑
  Admin Dashboard / CMS (optional)
```

---

## 🎯 Goals

- Store & version **channel-specific templates** (SMS, Email, Push, etc.)
- Support **variable interpolation**
- Allow **template versioning**, previews, rollback
- Serve **template content** at runtime (Notification Service will fetch it)
- Optional: **support multilingual templates**

---

## ⚙️ Functional Components

### 1. **Template Metadata**

- `id` (UUID)
- `name` (string, unique)
- `description` (text)
- `channel` (enum: `SMS`, `EMAIL`, `PUSH`, etc.)
- `language` (default: `en`)
- `version` (semver or incrementing int)
- `is_active` (bool)

### 2. **Template Body**

- `subject` (for email)
- `body` (main content, with placeholders like `{{user_name}}`)
- `fallback_text` (for devices/email clients that don’t support HTML, optional)

---

## 🧮 Sample JSON Template Entry

```json
{
  "id": "template-uuid",
  "name": "order_confirmation",
  "channel": "EMAIL",
  "language": "en",
  "version": 2,
  "is_active": true,
  "subject": "Order Confirmation for {{user_name}}",
  "body": "<h1>Hi {{user_name}}</h1><p>Your order {{order_id}} has been confirmed.</p>"
}
```

---

## 📦 API Design (Go Fiber)

### ✅ POST `/templates`

- Create a new version of a template
- Validates placeholders format
- Deactivates older version if `is_active = true`

### ✅ GET `/templates/:name/:channel?/:language?`

- Fetch latest active version
- Filters by name, channel, and language

### ✅ GET `/templates/:id`

- Get by template ID

### ✅ PUT `/templates/:id`

- Update template details (e.g., activate, deactivate)

### ✅ DELETE `/templates/:id`

- Soft delete a template version

---

## 🔁 Notification Service Integration

### 🧩 Flow

1. Notification Service calls `GET /templates/:name/:channel/:language`
2. Interpolates variables:

   ```go
   strings.ReplaceAll("Hello {{name}}", "{{name}}", user.Name)
   ```

3. Sends rendered content via SMS/Email/Push

---

## 🧵 Developer Responsibilities

| Task             | Description                                                 |
| ---------------- | ----------------------------------------------------------- |
| API Layer        | Create Go Fiber endpoints as per API spec                   |
| Validation       | Ensure placeholders use `{{variable}}` pattern              |
| Interpolation    | Write testable utility to interpolate placeholders          |
| Caching          | Add Redis or in-memory cache to avoid DB hit on every fetch |
| Version Control  | Implement version bumping & rollback logic                  |
| Language Support | Optionally allow multi-language fallback mechanism          |

---

## 🧰 DevOps Responsibilities

| Task           | Description                                            |
| -------------- | ------------------------------------------------------ |
| DB Setup       | PostgreSQL / YugabyteDB schema for templates           |
| Storage Backup | Setup cronjob to back up templates regularly           |
| RBAC           | Enable Role-Based Access for admin endpoints           |
| Secrets        | Use Kubernetes secrets for DB credentials              |
| Deployment     | Helm chart or K8s manifests                            |
| Monitoring     | Log structured access to templates using Logrus + LGTM |
| Caching Infra  | Optional Redis setup for template caching              |

---

## 🧮 Suggested DB Schema (PostgreSQL/Yugabyte)

```sql
CREATE TABLE templates (
  id UUID PRIMARY KEY,
  name TEXT NOT NULL,
  channel TEXT NOT NULL,
  language TEXT NOT NULL DEFAULT 'en',
  version INT NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  subject TEXT,
  body TEXT NOT NULL,
  fallback_text TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

Add indexes:

```sql
CREATE UNIQUE INDEX idx_active_template ON templates (name, channel, language, is_active)
  WHERE is_active = TRUE;
```

---

## 📐 Scaling Strategy

| Concern          | Approach                                              |
| ---------------- | ----------------------------------------------------- |
| High read volume | Cache rendered templates by (name, lang, channel)     |
| Frequent updates | Use Redis pub/sub to invalidate cache across replicas |
| Schema evolution | Use JSONB for storing dynamic template attributes     |
| Multilingual     | Add fallback language chain (e.g., `fr → en`)         |

---

## 🔒 Security & Validation

- Validate template content against invalid placeholders
- Add audit logs for template modifications
- Token-auth or mTLS for Notification Service to access templates
- Rate-limit write/update APIs

---

## 🧪 Testing Strategy

- Unit test template rendering and variable substitution
- E2E test with Notification Service
- Regression test version rollback scenarios
- Snapshot test for rendered output

---

## 🌐 Admin Dashboard (Future Scope)

If you want a CMS-like tool:

- CRUD templates with live preview
- Show rendered preview with test variables
- Version history and rollback button
- Built using React + Go Fiber backend
