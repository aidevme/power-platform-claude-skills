# File and Image Storage Testing

FakeXrmEasy includes an in-memory file storage system for testing plugins that work with file
and image columns. This allows you to test file upload, download, and manipulation logic without
connecting to a real Dataverse environment.

**Minimum Version Required:** FakeXrmEasy 2.6.0+ (Framework) or 3.6.0+ (.NET Core)

## Overview

File and image columns in Dataverse store binary data (PDFs, images, documents) referenced by
entity records. FakeXrmEasy simulates this with an in-memory file storage that:

- Stores uploaded files as binary data
- Validates file size, MIME types, and extensions
- Handles chunked uploads (for large files)
- Manages file lifecycle (create, update, delete)
- Supports both File columns and Image columns

**Read Microsoft's official documentation first:**
- [Files and Images Overview](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/files-images-overview)
- [File Column Data](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/file-column-data)
- [Image Column Data](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/image-column-data)

## File Storage Settings

### Default Settings

FakeXrmEasy uses default limits that can be overridden:

```csharp
[Fact]
public void Setup_Custom_File_Size_Limits()
{
    // ARRANGE: Override default file storage settings
    _context.SetProperty<IFileStorageSettings>(new FileStorageSettings
    {
        MaxSizeInKB = 5120,        // 5MB for files (default: varies)
        ImageMaxSizeInKB = 2048    // 2MB for images (default: varies)
    });
    
    // Now file uploads will be validated against these limits
}
```

### Column-Specific Limits via Metadata

Set limits per column using fake metadata:

```csharp
[Fact]
public void Setup_Column_Specific_File_Size()
{
    // ARRANGE: Create file attribute with specific size limit
    var fileAttributeMetadata = new FileAttributeMetadata
    {
        LogicalName = "dv_attachment",
        MaxSizeInKB = 1024  // 1MB limit for this specific column
    };
    
    var entityMetadata = new EntityMetadata
    {
        LogicalName = "dv_document"
    };
    entityMetadata.SetAttributeCollection(new List<AttributeMetadata>
    {
        fileAttributeMetadata
    });
    
    _context.InitializeMetadata(entityMetadata);
    
    // File uploads to dv_attachment will enforce 1MB limit
}
```

## Testing File Uploads

### Basic File Upload

```csharp
[Fact]
public void Should_Upload_File_To_File_Column()
{
    // ARRANGE
    var documentId = Guid.NewGuid();
    var document = new Entity("dv_document")
    {
        Id = documentId,
        ["dv_name"] = "My Document"
    };
    _context.Initialize(new[] { document });
    
    var fileContent = Encoding.UTF8.GetBytes("This is test file content");
    var fileBase64 = Convert.ToBase64String(fileContent);
    
    // ACT: Upload file using InitializeFileBlocksUploadRequest
    var initRequest = new InitializeFileBlocksUploadRequest
    {
        Target = new EntityReference("dv_document", documentId),
        FileAttributeName = "dv_file",
        FileName = "test-document.txt"
    };
    
    var initResponse = (InitializeFileBlocksUploadResponse)_service.Execute(initRequest);
    
    // Upload the file content in a single block
    var uploadRequest = new UploadBlockRequest
    {
        BlockId = Convert.ToBase64String(Encoding.UTF8.GetBytes("block1")),
        BlockData = fileBase64,
        FileContinuationToken = initResponse.FileContinuationToken
    };
    
    _service.Execute(uploadRequest);
    
    // Commit the upload
    var commitRequest = new CommitFileBlocksUploadRequest
    {
        BlockList = new[] { uploadRequest.BlockId },
        FileContinuationToken = initResponse.FileContinuationToken,
        FileName = "test-document.txt",
        MimeType = "text/plain"
    };
    
    var commitResponse = (CommitFileBlocksUploadResponse)_service.Execute(commitRequest);
    
    // ASSERT
    Assert.NotNull(commitResponse.FileId);
    
    // Verify file is associated with record
    var updated = _service.Retrieve("dv_document", documentId, new ColumnSet(true));
    Assert.True(updated.Contains("dv_file"));
}
```

### Simplified File Upload Helper

Create a helper method for common file upload scenarios:

```csharp
public class FileUploadTestHelper
{
    public static Guid UploadFile(
        IOrganizationService service,
        EntityReference target,
        string attributeName,
        string fileName,
        byte[] fileContent,
        string mimeType = "application/octet-stream")
    {
        var fileBase64 = Convert.ToBase64String(fileContent);
        
        // Initialize upload
        var initRequest = new InitializeFileBlocksUploadRequest
        {
            Target = target,
            FileAttributeName = attributeName,
            FileName = fileName
        };
        
        var initResponse = (InitializeFileBlocksUploadResponse)service.Execute(initRequest);
        
        // Upload single block
        var uploadRequest = new UploadBlockRequest
        {
            BlockId = Convert.ToBase64String(Encoding.UTF8.GetBytes("block1")),
            BlockData = fileBase64,
            FileContinuationToken = initResponse.FileContinuationToken
        };
        
        service.Execute(uploadRequest);
        
        // Commit upload
        var commitRequest = new CommitFileBlocksUploadRequest
        {
            BlockList = new[] { uploadRequest.BlockId },
            FileContinuationToken = initResponse.FileContinuationToken,
            FileName = fileName,
            MimeType = mimeType
        };
        
        var commitResponse = (CommitFileBlocksUploadResponse)service.Execute(commitRequest);
        
        return commitResponse.FileId;
    }
}

[Fact]
public void Upload_File_Using_Helper()
{
    // ARRANGE
    var documentId = Guid.NewGuid();
    _context.Initialize(new[] { new Entity("dv_document") { Id = documentId } });
    
    var fileContent = Encoding.UTF8.GetBytes("Test content");
    
    // ACT
    var fileId = FileUploadTestHelper.UploadFile(
        _service,
        new EntityReference("dv_document", documentId),
        "dv_file",
        "test.txt",
        fileContent,
        "text/plain"
    );
    
    // ASSERT
    Assert.NotEqual(Guid.Empty, fileId);
}
```

## Testing File Downloads

### Download File Content

```csharp
[Fact]
public void Should_Download_File_Content()
{
    // ARRANGE: Upload a file first
    var documentId = Guid.NewGuid();
    _context.Initialize(new[] { new Entity("dv_document") { Id = documentId } });
    
    var originalContent = "This is the file content to download";
    var fileContent = Encoding.UTF8.GetBytes(originalContent);
    
    var fileId = FileUploadTestHelper.UploadFile(
        _service,
        new EntityReference("dv_document", documentId),
        "dv_file",
        "download-test.txt",
        fileContent,
        "text/plain"
    );
    
    // ACT: Download the file
    var downloadRequest = new DownloadFileRequest
    {
        Target = new EntityReference("dv_document", documentId),
        FileAttributeName = "dv_file"
    };
    
    var downloadResponse = (DownloadFileResponse)_service.Execute(downloadRequest);
    
    // ASSERT
    Assert.NotNull(downloadResponse.Data);
    
    var downloadedBytes = Convert.FromBase64String(downloadResponse.Data);
    var downloadedContent = Encoding.UTF8.GetString(downloadedBytes);
    
    Assert.Equal(originalContent, downloadedContent);
    Assert.Equal("download-test.txt", downloadResponse.FileName);
    Assert.Equal("text/plain", downloadResponse.MimeType);
}
```

## Testing File Size Validation

### Exceed Maximum File Size

```csharp
[Fact]
public void Should_Throw_When_File_Exceeds_Max_Size()
{
    // ARRANGE: Set max file size to 1KB
    _context.SetProperty<IFileStorageSettings>(new FileStorageSettings
    {
        MaxSizeInKB = 1  // 1KB limit
    });
    
    var documentId = Guid.NewGuid();
    _context.Initialize(new[] { new Entity("dv_document") { Id = documentId } });
    
    // Create file larger than 1KB
    var largeContent = new byte[2048];  // 2KB
    new Random().NextBytes(largeContent);
    
    // ACT & ASSERT
    var ex = Assert.Throws<Exception>(() =>
    {
        FileUploadTestHelper.UploadFile(
            _service,
            new EntityReference("dv_document", documentId),
            "dv_file",
            "large-file.bin",
            largeContent
        );
    });
    
    Assert.Contains("size", ex.Message.ToLower());
}
```

## Testing Image Columns

### Upload Image

```csharp
[Fact]
public void Should_Upload_Image_To_Image_Column()
{
    // ARRANGE
    var contactId = Guid.NewGuid();
    _context.Initialize(new[] { new Contact { Id = contactId, FirstName = "John" } });
    
    // Create fake image data (1x1 pixel PNG)
    var imageBytes = Convert.FromBase64String(
        "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+M9QDwADhgGAWjR9awAAAABJRU5ErkJggg=="
    );
    
    // ACT: Upload image
    var fileId = FileUploadTestHelper.UploadFile(
        _service,
        new EntityReference("contact", contactId),
        "entityimage",
        "profile.png",
        imageBytes,
        "image/png"
    );
    
    // ASSERT
    Assert.NotEqual(Guid.Empty, fileId);
    
    var contact = _service.Retrieve("contact", contactId, new ColumnSet("entityimage"));
    Assert.True(contact.Contains("entityimage"));
}
```

### Download and Verify Image

```csharp
[Fact]
public void Should_Download_Image_With_Correct_Properties()
{
    // ARRANGE: Upload image
    var contactId = Guid.NewGuid();
    _context.Initialize(new[] { new Contact { Id = contactId } });
    
    var imageBytes = Convert.FromBase64String(
        "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+M9QDwADhgGAWjR9awAAAABJRU5ErkJggg=="
    );
    
    FileUploadTestHelper.UploadFile(
        _service,
        new EntityReference("contact", contactId),
        "entityimage",
        "avatar.png",
        imageBytes,
        "image/png"
    );
    
    // ACT: Download image
    var downloadRequest = new DownloadFileRequest
    {
        Target = new EntityReference("contact", contactId),
        FileAttributeName = "entityimage"
    };
    
    var response = (DownloadFileResponse)_service.Execute(downloadRequest);
    
    // ASSERT
    Assert.Equal("avatar.png", response.FileName);
    Assert.Equal("image/png", response.MimeType);
    
    var downloadedBytes = Convert.FromBase64String(response.Data);
    Assert.Equal(imageBytes.Length, downloadedBytes.Length);
}
```

## Testing File Deletion

### Delete File by Setting Column to Null

```csharp
[Fact]
public void Should_Delete_File_When_Column_Set_To_Null()
{
    // ARRANGE: Upload file
    var documentId = Guid.NewGuid();
    _context.Initialize(new[] { new Entity("dv_document") { Id = documentId } });
    
    var fileId = FileUploadTestHelper.UploadFile(
        _service,
        new EntityReference("dv_document", documentId),
        "dv_file",
        "to-delete.txt",
        Encoding.UTF8.GetBytes("Content"),
        "text/plain"
    );
    
    // ACT: Set file column to null
    var update = new Entity("dv_document") { Id = documentId };
    update["dv_file"] = null;
    _service.Update(update);
    
    // ASSERT: File should be removed
    var updated = _service.Retrieve("dv_document", documentId, new ColumnSet("dv_file"));
    Assert.False(updated.Contains("dv_file") && updated["dv_file"] != null);
}
```

## Testing Plugins with Files

### Plugin That Validates File Type

```csharp
public class FileValidationPlugin : IPlugin
{
    private static readonly string[] AllowedExtensions = { ".pdf", ".docx", ".txt" };
    
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        
        if (context.MessageName == "CommitFileBlocksUpload")
        {
            var fileName = context.InputParameters.Contains("FileName") 
                ? context.InputParameters["FileName"].ToString() 
                : string.Empty;
            
            var extension = Path.GetExtension(fileName).ToLower();
            
            if (!AllowedExtensions.Contains(extension))
            {
                throw new InvalidPluginExecutionException(
                    $"File type '{extension}' is not allowed. Allowed types: {string.Join(", ", AllowedExtensions)}"
                );
            }
        }
    }
}

[Fact]
public void Should_Reject_Invalid_File_Extension()
{
    // ARRANGE
    _context.RegisterPluginStep<FileValidationPlugin>(new PluginStepDefinition
    {
        MessageName = "CommitFileBlocksUpload",
        Stage = ProcessingStepStage.Prevalidation
    });
    
    var documentId = Guid.NewGuid();
    _context.Initialize(new[] { new Entity("dv_document") { Id = documentId } });
    
    var fileContent = Encoding.UTF8.GetBytes("Executable content");
    
    // ACT & ASSERT: Should reject .exe files
    var ex = Assert.Throws<InvalidPluginExecutionException>(() =>
    {
        FileUploadTestHelper.UploadFile(
            _service,
            new EntityReference("dv_document", documentId),
            "dv_file",
            "malware.exe",  // Not allowed
            fileContent
        );
    });
    
    Assert.Contains("not allowed", ex.Message);
}

[Fact]
public void Should_Accept_Valid_File_Extension()
{
    // ARRANGE
    _context.RegisterPluginStep<FileValidationPlugin>(new PluginStepDefinition
    {
        MessageName = "CommitFileBlocksUpload",
        Stage = ProcessingStepStage.Prevalidation
    });
    
    var documentId = Guid.NewGuid();
    _context.Initialize(new[] { new Entity("dv_document") { Id = documentId } });
    
    var fileContent = Encoding.UTF8.GetBytes("PDF content");
    
    // ACT: Should accept .pdf files
    var fileId = FileUploadTestHelper.UploadFile(
        _service,
        new EntityReference("dv_document", documentId),
        "dv_file",
        "document.pdf",  // Allowed
        fileContent,
        "application/pdf"
    );
    
    // ASSERT
    Assert.NotEqual(Guid.Empty, fileId);
}
```

### Plugin That Processes File Content

```csharp
public class FileScannerPlugin : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        var service = ((IOrganizationServiceFactory)serviceProvider.GetService(typeof(IOrganizationServiceFactory)))
            .CreateOrganizationService(context.UserId);
        
        if (context.MessageName == "CommitFileBlocksUpload")
        {
            var target = (EntityReference)context.InputParameters["Target"];
            var attributeName = context.InputParameters["FileAttributeName"].ToString();
            
            // Download the file that was just uploaded
            var downloadRequest = new DownloadFileRequest
            {
                Target = target,
                FileAttributeName = attributeName
            };
            
            var downloadResponse = (DownloadFileResponse)service.Execute(downloadRequest);
            var content = Encoding.UTF8.GetString(Convert.FromBase64String(downloadResponse.Data));
            
            // Scan content for forbidden words
            if (content.Contains("virus", StringComparison.OrdinalIgnoreCase))
            {
                throw new InvalidPluginExecutionException("File contains forbidden content");
            }
            
            // Update record with scan result
            var update = new Entity(target.LogicalName, target.Id);
            update["dv_scanneddate"] = DateTime.UtcNow;
            update["dv_scanstatus"] = "Clean";
            service.Update(update);
        }
    }
}

[Fact]
public void Should_Scan_File_Content_And_Update_Record()
{
    // ARRANGE
    _context.RegisterPluginStep<FileScannerPlugin>(new PluginStepDefinition
    {
        MessageName = "CommitFileBlocksUpload",
        Stage = ProcessingStepStage.Postoperation
    });
    
    var documentId = Guid.NewGuid();
    var document = new Entity("dv_document") 
    { 
        Id = documentId,
        ["dv_name"] = "Clean Document"
    };
    _context.Initialize(new[] { document });
    
    var cleanContent = Encoding.UTF8.GetBytes("This is a clean document");
    
    // ACT
    FileUploadTestHelper.UploadFile(
        _service,
        new EntityReference("dv_document", documentId),
        "dv_file",
        "clean.txt",
        cleanContent,
        "text/plain"
    );
    
    // ASSERT: Plugin should update scan status
    var updated = _service.Retrieve("dv_document", documentId, new ColumnSet(true));
    Assert.Equal("Clean", updated.GetAttributeValue<string>("dv_scanstatus"));
    Assert.True(updated.Contains("dv_scanneddate"));
}
```

## Initializing File Storage

Initialize the file storage with existing files for tests:

```csharp
[Fact]
public void Should_Initialize_File_Storage_With_Existing_Files()
{
    // ARRANGE: Create file storage initialization data
    var fileId = Guid.NewGuid();
    var fileContent = Encoding.UTF8.GetBytes("Existing file content");
    
    var fileInfo = new FakeXrmEasyFileInfo
    {
        FileId = fileId,
        FileName = "existing-file.txt",
        MimeType = "text/plain",
        Content = fileContent
    };
    
    _context.InitializeFileStorage(new[] { fileInfo });
    
    // Associate file with entity
    var documentId = Guid.NewGuid();
    var document = new Entity("dv_document") 
    { 
        Id = documentId,
        ["dv_file_fileid"] = fileId  // Reference to file
    };
    _context.Initialize(new[] { document });
    
    // ACT: Download the pre-existing file
    var downloadRequest = new DownloadFileRequest
    {
        Target = new EntityReference("dv_document", documentId),
        FileAttributeName = "dv_file"
    };
    
    var response = (DownloadFileResponse)_service.Execute(downloadRequest);
    
    // ASSERT
    var downloadedContent = Encoding.UTF8.GetString(Convert.FromBase64String(response.Data));
    Assert.Equal("Existing file content", downloadedContent);
}
```

## Best Practices

1. **Use helper methods** - File upload/download is verbose, create reusable helpers
2. **Test file size limits** - Verify both within and exceeding limits
3. **Test file types** - MIME types, extensions, allowed/blocked lists
4. **Clean up files** - Set columns to null or delete records to remove files
5. **Test chunked uploads** - For large files, test multi-block uploads
6. **Validate metadata** - Ensure file metadata (name, MIME, size) is correct

## Common Scenarios

### PDF Document Upload with Validation

```csharp
[Fact]
public void Should_Upload_PDF_And_Validate_Size()
{
    // ARRANGE: Set PDF size limit
    var pdfAttribute = new FileAttributeMetadata
    {
        LogicalName = "dv_pdfdocument",
        MaxSizeInKB = 5120  // 5MB
    };
    
    _context.InitializeMetadata(new EntityMetadata
    {
        LogicalName = "dv_contract"
    }.SetAttributeCollection(new[] { pdfAttribute }));
    
    var contractId = Guid.NewGuid();
    _context.Initialize(new[] { new Entity("dv_contract") { Id = contractId } });
    
    // Create 4MB PDF (within limit)
    var pdfContent = new byte[4 * 1024 * 1024];
    
    // ACT
    var fileId = FileUploadTestHelper.UploadFile(
        _service,
        new EntityReference("dv_contract", contractId),
        "dv_pdfdocument",
        "contract.pdf",
        pdfContent,
        "application/pdf"
    );
    
    // ASSERT
    Assert.NotEqual(Guid.Empty, fileId);
}
```
