# Veridoc — AI Document Authenticity Checker

A full-stack web app for verifying whether uploaded documents (PDF, DOCX,
JPG, JPEG, PNG) are authentic or potentially manipulated. Built as a
final-year engineering project with a modular backend so heuristic checks
can be swapped for real AI/ML models later without touching the API surface.

## Tech stack

- **Frontend:** React 18 + TypeScript + Tailwind CSS + Vite, React Router, Recharts, react-dropzone
- **Backend:** Node.js + Express
- **Database:** MongoDB (Mongoose)
- **Auth:** JWT (bcrypt-hashed passwords)
- **File storage:** Local disk (`backend/uploads`) — swappable for S3/GCS later
- **OCR:** Tesseract.js
- **PDF parsing/metadata:** pdf-lib, pdf-parse
- **PDF report generation:** PDFKit
- **Image forensics:** Jimp (error-level analysis), qrcode-reader

## Project structure

```
ai-doc-checker/
├── backend/
│   ├── config/db.js               # MongoDB connection
│   ├── models/                    # User, Document, Report schemas
│   ├── middleware/                # auth (JWT), upload (multer), error handler
│   ├── controllers/                # request handlers
│   ├── routes/                     # /api/auth, /api/documents, /api/reports
│   ├── services/
│   │   ├── aiAnalysisService.js    # orchestrates all analysis modules
│   │   ├── metadataService.js      # creation/modified date, author, software
│   │   ├── ocrService.js           # Tesseract OCR + PDF text extraction
│   │   ├── tamperDetectionService.js # error-level analysis (image forensics)
│   │   ├── signatureService.js     # digital signature field detection
│   │   ├── qrService.js            # QR/barcode decoding
│   │   ├── duplicateService.js     # exact hash + near-duplicate similarity
│   │   └── pdfReportService.js     # renders the downloadable PDF report
│   ├── utils/scoreCalculator.js    # risk points -> 0-100 score + risk label
│   └── server.js
└── frontend/
    └── src/
        ├── components/{layout,ui,landing,upload,report}/
        ├── pages/                  # Landing, Login, Signup, Dashboard, Upload, Report, History, Profile, Settings
        ├── context/                # AuthContext, ThemeContext
        ├── hooks/                  # useAuth, useTheme
        ├── services/api.ts         # Axios instance with JWT interceptor
        └── types/index.ts          # shared TS types mirroring backend schemas
```

## Getting started

### Prerequisites
- Node.js 18+
- A running MongoDB instance (local `mongod` or a free MongoDB Atlas cluster)

### 1. Backend

```bash
cd backend
cp .env.example .env
# edit .env: set MONGO_URI and a strong JWT_SECRET
npm install
npm run dev        # starts on http://localhost:5000
```

### 2. Frontend

```bash
cd frontend
npm install
npm run dev         # starts on http://localhost:5173
```

The Vite dev server proxies `/api` and `/uploads` requests to
`http://localhost:5000`, so no CORS configuration is needed in development.

### 3. Build for production

```bash
cd frontend && npm run build     # outputs static assets to frontend/dist
cd backend && npm start           # serve the API (put dist/ behind nginx or similar)
```

## How the analysis pipeline works

`services/aiAnalysisService.js` is the single orchestrator. On each analyze
request it runs, in parallel:

1. **Metadata analysis** — creation/modification dates, author, producing
   software; flags editing-software fingerprints and suspicious date gaps.
2. **OCR / text extraction** — Tesseract.js for images, `pdf-parse` for PDF
   text layers; flags low OCR confidence and irregular spacing.
3. **Tamper detection** — error-level analysis (re-compression diffing) for
   images; structural checks for PDFs.
4. **Signature verification** — detects embedded PDF signature fields.
5. **QR/barcode scanning** — decodes any QR code found in an image.
6. **Duplicate detection** — SHA-256 exact-match plus Jaccard similarity
   over extracted text for near-duplicates.

Each module returns `{ findings[], riskPoints }`. The orchestrator sums risk
points, converts them into a 0–100 **authenticity score** and a **Low /
Medium / High** risk label (`utils/scoreCalculator.js`), and writes a
`Report` document.

## Swapping in real AI models later

Every service file has a documented extension point:

- `tamperDetectionService.js` → `analyzeWithRealModel()` — plug in a
  trained forgery-detection CNN or a hosted vision API.
- `signatureService.js` → `verifySignatureCrypto()` — implement real
  PKCS#7/X.509 certificate chain validation (e.g. with `node-forge`).
- `aiAnalysisService.js` → `generateAiSummary()` — replace the rule-based
  summary with a call to a hosted LLM (Claude, GPT, etc.) for a natural-
  language explanation of the findings.
- `metadataService.js` → `extractDocxMetadata()` — currently a stub;
  unzip the `.docx` and parse `docProps/core.xml`.
- `duplicateService.js` — swap text-shingling for embedding-based vector
  similarity search for more robust near-duplicate detection.

Because the rest of the app (routes, controllers, frontend) only depends on
the `{ findings, riskPoints }` contract, none of that code needs to change
when a module's internals are upgraded.

## Notes & limitations

- This is a heuristic-based analysis pipeline intended for a student/
  demonstration project, not a certified forensic tool — the UI and reports
  say so explicitly.
- File storage is local disk; for production, swap `middleware/upload.js`
  for an S3/GCS-backed multer storage engine.
- Rasterizing scanned PDF pages for OCR (rather than only reading the text
  layer) is left as an extension point — wire in `pdf.js` or `pdf-poppler`
  to render pages to images before calling `ocrService`.
