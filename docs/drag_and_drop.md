Drag & Drop (File Upload) — Implementation Guide
=================================================

This document explains how drag-and-drop file upload is implemented in this project (`app/components/FileUploader.tsx`), how it integrates with the rest of the app, and recommended improvements and tests.

Overview
--------
The project uses the react-dropzone library to implement a simple, accessible drag-and-drop area that accepts a single PDF file up to 20MB. The component is intentionally small and focused: it handles user selection and basic validation, then passes the selected File object to a parent component via a callback prop `onFileSelect`.

Where to find the code
----------------------
- Component: `app/components/FileUploader.tsx`
- Small helper used to display file sizes: `app/lib/utils.ts` (function `formatSize`)
- PDF parsing/preview integration point: `app/lib/pdf2img.ts` (used by other components after a file is selected)

Key implementation details
-------------------------
1. react-dropzone
- The component imports and uses `useDropzone` from `react-dropzone`.
- `useDropzone` returns helpers used to wire up the drop area:
  - `getRootProps()` — props to spread on the root container (handles drag events)
  - `getInputProps()` — props to spread on a hidden <input> (handles file selection via dialog)
  - `acceptedFiles` — list of accepted files (driven by accept/maxSize/multiple options)
  - `isDragActive` — boolean to render UI when a file is dragged over the area

2. Accept & Validation
- The hook is configured with:
  - `multiple: false` — only one file allowed
  - `accept: { 'application/pdf': ['.pdf'] }` — only PDF files
  - `maxSize: 20 * 1024 * 1024` — 20MB maximum file size
- `react-dropzone` performs validation automatically. The current component reads `acceptedFiles` but does not process `fileRejections` — see "Improvements" below.

3. onDrop callback
- `onDrop` is defined with `useCallback` and receives an array of accepted files.
- The component picks the first file (or null) and calls `onFileSelect?.(file)`.
- The parent component receives this File object and is expected to handle the next steps: parsing, analysis, previews, uploading to server, etc.

4. Selected file UI
- If a file is present in `acceptedFiles` the component shows a small preview row:
  - PDF icon
  - File name (truncated) and human-readable size (via `formatSize`)
  - A remove button that calls `onFileSelect?.(null)` to clear selection
- Otherwise, it shows the default prompt: "Click to upload or drag and drop" and notes the accepted file type and max size.

5. Integration points
- Parsing: After the parent receives the File object, the app typically uses `pdf2img.ts` or other helpers to create previews and extract text using `pdfjs-dist`.
- Analysis: The File/text then moves through the AI pipeline for section detection and scoring.

Usage (example)
----------------
A simple parent component that uses `FileUploader` might look like this:

```tsx
import { useState } from 'react'
import FileUploader from '../components/FileUploader'
import { parsePdfAndAnalyze } from '../lib/pdf2img'

function UploadPage() {
  const [file, setFile] = useState<File | null>(null)
  const [loading, setLoading] = useState(false)

  async function handleFileSelect(f: File | null) {
    setFile(f)
    if (!f) return

    setLoading(true)
    try {
      // Example helper: extract text or create preview images
      const data = await parsePdfAndAnalyze(f)
      // send `data` to AI pipeline or update UI
    } finally {
      setLoading(false)
    }
  }

  return (
    <div>
      <FileUploader onFileSelect={handleFileSelect} />
      {loading && <p>Processing...</p>}
    </div>
  )
}
```

Accessibility
-------------
- `react-dropzone` provides keyboard support via the underlying file input. The component spreads `getRootProps()` which includes `tabIndex` and event handlers.
- Ensure the root drop area has descriptive text, which the component already provides ("Click to upload or drag and drop").
- For screen readers, consider adding `aria-label` or `aria-describedby` on the root container.

Testing the behavior locally
----------------------------
1. Start the app locally (Vite / react-router dev script in this project):

```powershell
npm run dev
```

2. Open the Upload page and try:
- Dragging a PDF file onto the drop area
- Clicking the area to open the file picker and selecting a PDF
- Selecting a non-PDF file (should be rejected by `react-dropzone`)
- Selecting a file larger than 20MB (should be rejected)

3. Observe UI reactions (file preview, remove button). If parsing is wired, the parent should start processing the file.

Limitations & improvements
--------------------------
The current implementation is intentionally minimal. Suggested improvements:

1. Handle rejected files (`fileRejections`)
   - Present clear error messages to the user (wrong type or too large).
   - Example: show a small inline alert explaining why the file was rejected.

2. Show progress & validations
   - Show a small progress bar when processing/reading the file.
   - Disable re-drop while processing.

3. Accessibility enhancements
   - Add `aria-live` region to announce validation errors.
   - Add keyboard-only cues for dropping a file.

4. Image/preview generation
   - Add inline previews using `pdf2img.ts` to render the first page as an image.
   - Cache generated previews for faster repeat viewing.

5. Multiple formats & conversions
   - Allow DOCX uploads by converting them to PDF client-side or via a backend service.
   - Use `accept` to expand accepted MIME types as needed.

6. Unit & e2e tests
   - Unit test the component logic (callback firing, UI rendering) using React Testing Library.
   - E2E test the drag-and-drop flow with Playwright or Cypress (drag-and-drop can be simulated via the file input).

Example: add basic rejection handling
------------------------------------
Below is a short snippet showing how to capture rejected files and display errors with `useDropzone`.

```tsx
const {getRootProps, getInputProps, acceptedFiles, fileRejections} = useDropzone({
  onDrop,
  multiple: false,
  accept: { 'application/pdf': ['.pdf']},
  maxSize: maxFileSize,
})

// Render rejection messages
fileRejections.map(rejection => (
  <div key={rejection.file.name} className="text-red-600">
    {rejection.file.name} - {rejection.errors.map(e => e.message).join(', ')}
  </div>
))
```

Conclusion
----------
`FileUploader.tsx` implements a small, effective drag-and-drop area for PDF resumes using `react-dropzone`. The component delegates actual parsing and AI analysis to parent components and helper libraries. The current implementation handles the happy path well; adding user-friendly error messages, previews, accessibility tweaks, tests, and support for other formats would make it production-ready.

If you'd like, I can:
- Update `FileUploader.tsx` to show file rejections and inline errors.
- Add a preview generation flow that wires `pdf2img.ts` to show the first-page image.
- Create unit tests for the uploader using React Testing Library.

Which improvement should I implement next?