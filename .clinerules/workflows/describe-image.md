# Describe image

1. When user ask to describe an image. Use `read_file` tool to get the base64 data of the image.

```xml
<read_file>
<path>path/to/file</path>
</read_file>
```

2. Analyze the image and respond with a short description of the image.
