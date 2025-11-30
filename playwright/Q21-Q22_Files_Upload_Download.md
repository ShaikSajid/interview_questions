# Playwright Interview Questions: Q21-Q22

## Q21: How do you handle file uploads in Playwright?

### Answer:

Playwright supports various file upload methods including input elements, drag-and-drop, and buffer uploads.

### 1. File Upload Methods

```typescript
import { test, expect } from '@playwright/test';

test('upload single file', async ({ page }) => {
  await page.goto('https://example.com/upload');
  
  // Set files on input element
  await page.locator('input[type="file"]').setInputFiles('path/to/file.pdf');
  
  await page.locator('#upload-btn').click();
  await expect(page.locator('.success')).toBeVisible();
});

test('upload multiple files', async ({ page }) => {
  await page.goto('https://example.com/upload');
  
  await page.locator('input[type="file"]').setInputFiles([
    'file1.jpg',
    'file2.jpg',
    'file3.jpg'
  ]);
});

test('upload from buffer', async ({ page }) => {
  await page.goto('https://example.com/upload');
  
  await page.locator('input[type="file"]').setInputFiles({
    name: 'test.txt',
    mimeType: 'text/plain',
    buffer: Buffer.from('Test content')
  });
});
```

### 2. Drag and Drop Upload

```typescript
test('drag and drop file', async ({ page }) => {
  await page.goto('https://example.com/upload');
  
  const dataTransfer = await page.evaluateHandle(() => new DataTransfer());
  
  await page.locator('input[type="file"]').setInputFiles('file.pdf');
  
  // Simulate drag and drop
  await page.dispatchEvent('.drop-zone', 'drop', { dataTransfer });
});
```

### 3. Real-World Scenario

```typescript
test('document upload flow', async ({ page }) => {
  await page.goto('https://app.example.com/documents');
  
  // Upload document
  await page.locator('input#document').setInputFiles('resume.pdf');
  
  // Fill metadata
  await page.locator('#doc-title').fill('My Resume');
  await page.selectOption('#category', 'personal');
  
  // Submit
  await page.locator('#submit').click();
  
  // Wait for upload progress
  await expect(page.locator('.upload-progress')).toBeVisible();
  await expect(page.locator('.upload-complete')).toBeVisible({ timeout: 30000 });
  
  // Verify uploaded
  await expect(page.locator('.document-list')).toContainText('My Resume');
});
```

### Related Topics
- **Q22**: File downloads
- **Q8**: Form interactions

---

## Q22: How do you handle file downloads?

### Answer:

Playwright provides built-in download handling with methods to wait for downloads and access downloaded files.

### 1. Download Methods

```typescript
test('download file', async ({ page }) => {
  await page.goto('https://example.com/downloads');
  
  // Start waiting for download before clicking
  const downloadPromise = page.waitForEvent('download');
  await page.locator('#download-btn').click();
  const download = await downloadPromise;
  
  // Get download properties
  console.log(download.suggestedFilename());
  
  // Save to disk
  await download.saveAs('downloads/' + download.suggestedFilename());
  
  // Or get as stream
  const stream = await download.createReadStream();
});

test('verify download content', async ({ page }) => {
  await page.goto('https://example.com');
  
  const downloadPromise = page.waitForEvent('download');
  await page.locator('#download-report').click();
  const download = await downloadPromise;
  
  // Save and verify
  const path = 'temp/' + download.suggestedFilename();
  await download.saveAs(path);
  
  // Read and verify content
  const fs = require('fs');
  const content = fs.readFileSync(path, 'utf-8');
  expect(content).toContain('Report Data');
});
```

### 2. Multiple Downloads

```typescript
test('handle multiple downloads', async ({ page }) => {
  await page.goto('https://example.com/downloads');
  
  const downloads = [];
  
  page.on('download', download => {
    downloads.push(download);
  });
  
  // Trigger multiple downloads
  await page.locator('.download-all').click();
  
  // Wait for all downloads
  await page.waitForTimeout(5000);
  
  expect(downloads).toHaveLength(3);
  
  // Save all
  for (const download of downloads) {
    await download.saveAs('downloads/' + download.suggestedFilename());
  }
});
```

### 3. Real-World Scenario: Report Export

```typescript
test('export and verify report', async ({ page }) => {
  await page.goto('https://dashboard.example.com');
  
  // Apply filters
  await page.selectOption('#date-range', 'last-30-days');
  await page.selectOption('#format', 'pdf');
  
  // Export
  const downloadPromise = page.waitForEvent('download');
  await page.locator('#export-btn').click();
  
  const download = await downloadPromise;
  expect(download.suggestedFilename()).toMatch(/report.*\.pdf$/);
  
  const path = 'downloads/report.pdf';
  await download.saveAs(path);
  
  // Verify file exists and size
  const fs = require('fs');
  const stats = fs.statSync(path);
  expect(stats.size).toBeGreaterThan(1024); // At least 1KB
});
```

### Related Topics
- **Q21**: File uploads
- **Q14**: Network interception

---

*Document Version: 1.0*  
*Last Updated: November 2025*
