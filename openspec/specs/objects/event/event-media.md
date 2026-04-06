# Objects: EventImage & EventFile

**Module:** event

---

## EventImage

**Table:** `event_images` | **PK:** Long

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-increment |
| event_id | UUID | FK → events.id, NOT NULL |
| url | String | NOT NULL |
| idx | Integer | NOT NULL |

- `idx` controls display order
- Cover image is stored as `cover_url` on the Event entity itself
- Gallery images are stored here

---

## EventFile

**Table:** `event_files` | **PK:** Long

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-increment |
| event_id | UUID | FK → events.id, NOT NULL |
| file_name | String | nullable |
| url | String | S3/MinIO URL |
| content_type | String | nullable |
| file_size | Long | nullable |
| uploaded_at | LocalDateTime | auto |

- Attachments (PDF, DOCX) for event documentation
- Max file size: 20 MB (`app.upload.max-file-size`)
- Allowed types: PDF, DOC, DOCX
