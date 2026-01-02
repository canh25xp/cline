# Analyze image with Cline

There's two main way attach an image to Cline.

1. Using the ["Add Files & Images"](../webview-ui/src/components/chat/ChatTextArea.tsx) button below the chat box.
   This way you directly, you directly select and add the image file and attach it to Cline.
2. Using the [`read_file`](src/core/prompts/system-prompt/tools/read_file.ts) tools.
   This way, you indirectly tell Cline to use the `read_file` tool on the image file path that you specify.

Although the implementation for each method is different, they both doing somewhat the same thing.
In fact, I believe both method can be refactor using 1 file handler.

---

When the user presses the "Add Files & Images" button in Cline, here's the complete flow of what happens:

## 1. UI Layer (Webview)

- The button is a `VSCodeButton` with `data-testid="files-button"` in `ChatTextArea.tsx`
- When clicked, it calls `onSelectFilesAndImages()` prop function (only if `shouldDisableFilesAndImages` is false)
- This prop is passed from `ChatView.tsx` as `selectFilesAndImages` function

## 2. ChatView Component

- The `selectFilesAndImages` function calls `FileServiceClient.selectFiles(BooleanRequest.create({ value: selectedModelInfo.supportsImages }))`
- This is a gRPC client call that communicates with the backend extension

## 3. Backend Extension (gRPC Service)

- The gRPC service routes the call to `src/core/controller/file/selectFiles.ts`
- This controller function calls `selectFilesIntegration(request.value)` from `@integrations/misc/process-files`

## 4. File Selection Integration

- The `selectFiles` function in `src/integrations/misc/process-files.ts` handles the actual file selection:
  - Shows a file dialog using `HostProvider.window.showOpenDialogue()`
  - Allows selection of multiple files with filters based on whether images are supported
  - Supported image extensions: `png`, `jpg`, `jpeg`, `webp` (7500px max dimensions)
  - Supported other file extensions: `xml`, `json`, `txt`, `log`, `md`, `docx`, `ipynb`, `pdf`, `xlsx`, `csv` (20MB max size)
  - Processes selected files:
    - Images: Reads file, checks dimensions, converts to base64 data URLs with proper MIME types
    - Other files: Checks file size, returns file paths
  - Returns an object with `{ images: string[], files: string[] }`

## 5. Response Handling

- The controller wraps the result in `StringArrays.create({ values1: images, values2: files })`
- `values1` contains image data URLs, `values2` contains file paths
- The gRPC response is sent back to the webview

## 6. Webview Update

- The `ChatView.tsx` receives the response and updates state:
  - Images are added to `selectedImages` state (as data URLs)
  - Files are added to `selectedFiles` state (as file paths)
  - Thumbnails component displays previews of selected files/images
  - The files/images are now ready to be sent with the next chat message

The button essentially opens a file picker dialog, processes the selected files (converting images to data URLs and validating file sizes), and stores them in the component state for inclusion in the next message to the AI.

---

# READ_FILE Operation Implementation in Cline

Implementation of the READ_FILE operation:

## 1. Tool Definition and Prompt

**File:** `src/core/prompts/system-prompt/tools/read_file.ts`

- Defines the READ_FILE tool specification for different model families
- Specifies the tool description, parameters (path, task_progress), and usage instructions
- Tool ID: `ClineDefaultTool.FILE_READ`

## 2. Tool Execution Flow

**File:** `src/core/task/ToolExecutor.ts`

- The `ToolExecutor` class manages all tool execution
- READ_FILE handler is registered in the `registerToolHandlers()` method
- Routes tool execution through the `ToolExecutorCoordinator`

## 3. Main Handler Implementation

**File:** `src/core/task/tools/handlers/ReadFileToolHandler.ts`

- **Class:** `ReadFileToolHandler` implements `IFullyManagedTool`
- **Key Responsibilities:**
  - Parameter validation (path is required)
  - Cline ignore path checking
  - Workspace path resolution (multi-root support)
  - Approval flow handling (auto-approval vs manual approval)
  - Telemetry capture
  - PreToolUse hook execution
  - File content extraction via `extractFileContent`

## 4. File Content Extraction

**File:** `src/integrations/misc/extract-file-content.ts`

- **Function:** `extractFileContent(absolutePath: string, modelSupportsImages: boolean)`
- **Logic:**
  - Checks if file exists
  - Determines file type by extension
  - Handles images (.png, .jpg, .jpeg, .webp) if model supports images
  - Delegates text files to `callTextExtractionFunctions`

## 5. Text File Processing

**File:** `src/integrations/misc/extract-text.ts`

- **Function:** `callTextExtractionFunctions(filePath: string)`
- **Special File Types:**
  - **PDF:** Uses `pdf-parse` library to extract text
  - **DOCX:** Uses `mammoth` library to extract raw text
  - **IPYNB:** Parses JSON and extracts code/markdown cell content
  - **XLSX:** Uses `exceljs` to read and format Excel content
- **Regular Text Files:**
  - Reads file into buffer
  - Detects encoding using `jschardet`
  - Decodes with `iconv-lite`
  - 20MB file size limit

## 6. Image Processing

**File:** `src/integrations/misc/extract-images.ts`

- **Function:** `extractImageContent(filePath: string)`
- **Process:**
  - Reads file into buffer
  - Validates image dimensions (max 7500x7500px)
  - Converts to base64
  - Creates Anthropic image block format
  - Returns image block for AI processing

## 7. Key Features

- **Multi-root Workspace Support:** Resolves paths across multiple workspace roots
- **Auto-approval System:** Respects user auto-approval settings
- **Cline Ignore Integration:** Respects `.clineignore` rules
- **Telemetry:** Captures tool usage metrics
- **Error Handling:** Comprehensive error handling with user feedback
- **Hook Support:** PreToolUse and PostToolUse hook integration
- **Focus Chain:** Integration with focus chain tracking
- **Large File Protection:** 20MB limit for text files

## 8. Execution Flow Summary

1. AI requests READ_FILE with path parameter
2. `ReadFileToolHandler` validates and processes the request
3. Approval flow (auto or manual) is handled
4. File path is resolved in workspace context
5. `extractFileContent` determines file type and processing method
6. Content is extracted (text parsing or image conversion)
7. Result is returned to AI with proper formatting
8. File context is tracked for focus chain
9. Telemetry is captured

---

Long story short, either you use that little plus button to add file or tell Cline to read it, it returns the converted image in `base64` format along with the mimeType.

```ts
// process-files.ts
const base64 = buffer.toString("base64");
const mimeType = getMimeType(filePath);
return { type: "image", data: `data:${mimeType};base64,${base64}` };

// extract-images.ts
const base64 = buffer.toString("base64");
const mimeType = getMimeType(filePath) as
  | "image/jpeg"
  | "image/png"
  | "image/webp";

const imageBlock: Anthropic.ImageBlockParam = {
  type: "image",
  source: {
    type: "base64",
    media_type: mimeType,
    data: base64,
  },
};

return { success: true, imageBlock };
```
