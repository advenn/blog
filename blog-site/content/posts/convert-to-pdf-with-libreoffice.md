---
date: '2025-11-21T23:42:51+01:00'
draft: false
title: 'Converting Wide Excel Tables to Single-Page PDFs with LibreOffice'
tags: ['libreoffice', 'pdf', 'python', 'docker', 'automation']
---

## The Problem

While working on a freelance project, I needed to generate PDF reports from Excel files. The challenge? The Excel files were wide - around 15 columns containing employee data like ID, name, email, department, job title, hire date, salary, and more.

I used LibreOffice's CLI utility `convert-to` in headless mode to convert the files. The conversion worked, but the result was unusable:

![Wide Excel table with many columns](/photos/long-excel-table.png)

When converted to PDF without any special options, LibreOffice split the columns across multiple pages - the first 3 columns on page 1, the next 4 on page 2, and so on. If the Excel file had more rows than could fit on a single page, things got even messier with both horizontal and vertical splits.

![Table split across multiple pages](/photos/table-split-horizontally.jpg)

This made the PDF reports unreadable and unprofessional.

## The Solution

After digging through LibreOffice documentation, I found a conversion option that solves this problem: `SinglePageSheets`. This option forces each Excel sheet to fit onto a single PDF page, scaling the content appropriately.

![All data on a single page](/photos/full-table-single-page.jpg)

The key is using the right export filter:

```bash
libreoffice --headless --convert-to \
  'pdf:calc_pdf_Export:{"SinglePageSheets":{"type":"boolean","value":"true"}}' \
  --outdir /output input.xlsx
```

This command tells LibreOffice to:
- Run in headless mode (no GUI)
- Convert to PDF using the Calc PDF export filter
- Enable the `SinglePageSheets` option to fit the entire sheet on one page

## Docker Implementation

I've created a complete Docker setup that wraps this functionality in a simple FastAPI web service. You can find the full source code on GitHub: [https://github.com/advenn/libreoffice_sample](https://github.com/advenn/libreoffice_sample)

### Dockerfile Breakdown

```dockerfile
# Use Ubuntu as the base image
FROM ubuntu:22.04

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Set working directory
WORKDIR /app
```

These lines set up the base image and configure Python to not write bytecode files and to output directly to the console.

```dockerfile
# Install system dependencies for LibreOffice
RUN apt-get update && apt-get install -y \
    wget \
    curl \
    gdebi-core \
    tar \
    xz-utils \
    libfontconfig1 \
    libxrender1 \
    libxext6 \
    libglx0 \
    libdbus-glib-1-2 \
    fonts-dejavu \
    python3 \
    python3-pip \
    libsm6 \
    libxinerama1 \
    libxrandr2 \
    libxcursor1 \
    libpng16-16 \
    libfreetype6 \
    libreoffice-core \  
    --no-install-recommends && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

This installs all required dependencies:
- Download utilities (`wget`, `curl`)
- Package handling (`gdebi-core`, `tar`, `xz-utils`)
- Graphics libraries required by LibreOffice
- Fonts for proper text rendering
- Python 3 and pip
- LibreOffice core package (as a base), the manual installation won't work without core package

```dockerfile
# Copy and run the LibreOffice installation script
COPY scripts/install_libre.sh /tmp/install_libre.sh
RUN chmod +x /tmp/install_libre.sh && /tmp/install_libre.sh && rm /tmp/install_libre.sh
```

The installation script automatically downloads the latest stable version of LibreOffice from the Document Foundation and installs it. This ensures you always get the most recent version with all features.

```dockerfile
# Create necessary directories
RUN mkdir -p /app/pdf

# Expose port 8000 for the FastAPI app
EXPOSE 8000

# Run FastAPI with uvicorn
CMD ["python3", "-m", "uvicorn", "web.app:app", "--host", "0.0.0.0", "--port", "8000"]
```

Finally, we create the output directory and start the FastAPI server.

### The Conversion Code

The core conversion happens in `utils/convert_to_pdf.py`:

```python
def excel2pdf_via_libreoffice(
        excel_file: Path,
        output_dir: Path = RENDERED_FILES_DIR / "pdfs",
        single_page: bool = True
) -> Optional[Path]:
    # Define the export filter based on single_page option
    if single_page:
        export_filter = 'pdf:calc_pdf_Export:{"SinglePageSheets":{"type":"boolean","value":"true"}}'
    else:
        export_filter = 'pdf'

    # Get the LibreOffice command dynamically
    libreoffice_cmd = get_libreoffice_command()

    # Build command
    cmd = [
        libreoffice_cmd,
        "--headless",
        "--convert-to", export_filter,
        "--outdir", str(output_dir),
        str(excel_file)
    ]

    # Run the command
    result = subprocess.run(cmd, capture_output=True, text=True)
```

The `single_page` parameter controls whether to use the `SinglePageSheets` option. The code dynamically detects which LibreOffice command is available (since different installations may have different version-specific commands like `libreoffice25.2` or `libreoffice7.6`).

### FastAPI Web Interface

The web interface (`web/app.py`) provides a simple form where you can:
1. Upload an Excel file
2. Toggle the "single page conversion" checkbox
3. Download the converted PDF

```python
@app.post("/")
async def to_pdf(
    file: UploadFile = File(...),
    single_page: str = Form(default="")
):
    # Save uploaded file
    with tempfile.NamedTemporaryFile(delete=False, suffix=Path(file.filename).suffix) as tmp:
        content = await file.read()
        tmp.write(content)
        tmp_path = Path(tmp.name)

    # Convert to PDF with single_page option
    use_single_page = single_page == "true"
    pdf_path = excel2pdf_via_libreoffice(tmp_path, single_page=use_single_page)

    return FileResponse(pdf_path, filename=f"{tmp_path.stem}.pdf")
```

## Building and Running

Clone the repository and build the Docker image:

```bash
git clone https://github.com/advenn/libreoffice_sample
cd libreoffice_sample
docker build -t libreoffice-pdf-converter .
```

Run the container:

```bash
docker run -p 8000:8000 libreoffice-pdf-converter
```

Or use docker-compose:

```bash
docker-compose up
```

Then open your browser to `http://localhost:8000` and upload an Excel file to convert it to PDF.

## Conclusion

The `SinglePageSheets` option in LibreOffice's PDF export filter is a simple but effective solution for converting wide Excel tables to readable single-page PDFs. This is particularly useful for generating reports where you need all data visible at once without awkward page splits.

The Docker implementation makes it easy to deploy this as a microservice in your infrastructure or run it locally for quick conversions.
