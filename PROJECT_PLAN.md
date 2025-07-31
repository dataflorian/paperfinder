# Scientific Paper Downloader - Implementation Plan

## Project Overview
A Python application that downloads full PDFs of scientific papers based on information provided in a CSV file. The system uses multiple approaches to locate and download papers, starting with Google Scholar and falling back to Sci-Hub when necessary.

## Development Environment Setup

### Package Management with uv
- **Package Manager**: uv for fast dependency management
- **Virtual Environment**: uv venv for isolated development
- **Dependency Locking**: uv lock for reproducible builds
- **Installation**: `uv pip install -r requirements.txt`

### Project Structure
```
paperfinder/
├── src/
│   ├── __init__.py
│   ├── csv_processor.py
│   ├── google_scholar_searcher.py
│   ├── scihub_searcher.py
│   ├── search_strategy.py
│   ├── download_manager.py
│   ├── file_validator.py
│   ├── rate_limiter.py
│   ├── logger.py
│   ├── report_generator.py
│   ├── config.py
│   └── main.py
├── tests/
│   ├── __init__.py
│   ├── test_csv_processor.py
│   ├── test_google_scholar.py
│   └── test_scihub.py
├── downloads/
│   ├── successful/
│   ├── failed/
│   └── logs/
├── requirements.txt
├── pyproject.toml
├── .env.example
└── README.md
```

## Core Implementation Requirements

### 1. CSV Input Processing
**File**: `src/csv_processor.py`
- **Input Validation**: 
  - Required columns: DOI, title
  - Optional columns: authors, year, journal
  - Encoding detection (UTF-8, ISO-8859-1)
  - Duplicate DOI detection
- **Data Cleaning**:
  - Strip whitespace from all fields
  - Normalize DOI format (remove http/https prefixes)
  - Handle missing values gracefully
- **Output**: Pandas DataFrame with validated data

### 2. Google Scholar Search Implementation
**File**: `src/google_scholar_searcher.py`

#### Rate Limiting Strategy
- **Request Limits**: Maximum 10 requests per minute
- **Delay Between Requests**: 6-8 seconds minimum
- **Session Management**: Rotate user agents every 50 requests
- **Blocking Detection**: Monitor for CAPTCHA or 429 responses
- **Recovery**: Exponential backoff (30s, 60s, 120s) on blocking

#### Search Efficiency Techniques
- **Primary Search**: Direct DOI search in quotes: `"10.1038/nature12345"`
- **Fallback Search**: Title + first author: `"Paper Title" "Author Name"`
- **Result Filtering**: 
  - Check for PDF links in search results
  - Verify link accessibility before download attempt
  - Prioritize direct PDF links over repository links
- **Caching**: Store successful search results to avoid re-searching

#### Implementation Details
```python
# Rate limiting configuration
GOOGLE_SCHOLAR_CONFIG = {
    'requests_per_minute': 10,
    'delay_between_requests': 7,
    'user_agents': [
        'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
        'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
        # Add more user agents
    ],
    'max_retries': 3,
    'backoff_factor': 2
}
```

### 3. Sci-Hub Integration
**File**: `src/scihub_searcher.py`
- **Mirror Selection**: Dynamic mirror availability checking
- **DOI Processing**: Direct DOI to Sci-Hub URL conversion
- **Download Strategy**: 
  - Try primary Sci-Hub domain first
  - Fallback to working mirrors
  - Handle redirects and anti-bot measures
- **Rate Limiting**: 5 requests per minute (more conservative)

### 4. Download Management
**File**: `src/download_manager.py`
- **File Organization**:
  - `downloads/successful/{year}/{month}/`
  - `downloads/failed/` with error logs
- **Naming Convention**: `{DOI}_{sanitized_title}.pdf`
- **Duplicate Detection**: SHA-256 hash checking
- **Resume Capability**: Partial download recovery
- **File Validation**: PDF header verification, minimum size check

### 5. Configuration Management
**File**: `src/config.py`
```python
# Configuration structure
CONFIG = {
    'search': {
        'google_scholar': {
            'enabled': True,
            'requests_per_minute': 10,
            'delay_between_requests': 7,
            'max_retries': 3
        },
        'scihub': {
            'enabled': True,
            'requests_per_minute': 5,
            'delay_between_requests': 12,
            'max_retries': 2
        }
    },
    'download': {
        'timeout': 30,
        'max_file_size': 50 * 1024 * 1024,  # 50MB
        'min_file_size': 1024,  # 1KB
        'chunk_size': 8192
    },
    'logging': {
        'level': 'INFO',
        'file': 'downloads/logs/paperfinder.log',
        'max_size': 10 * 1024 * 1024,  # 10MB
        'backup_count': 5
    }
}
```

## Dependencies (requirements.txt)

### Core Dependencies
```
pandas>=2.0.0
requests>=2.31.0
beautifulsoup4>=4.12.0
tqdm>=4.65.0
python-dotenv>=1.0.0
```

### Search Dependencies
```
scholarly>=1.7.11
scihub-downloader>=0.1.0
```

### Optional Dependencies
```
PyPDF2>=3.0.0
aiohttp>=3.8.0
selenium>=4.10.0
```

## Implementation Phases

### Phase 1: Foundation Setup
- [ ] Initialize project with uv: `uv init paperfinder`
- [ ] Create project structure and directories
- [ ] Set up virtual environment: `uv venv`
- [ ] Install dependencies: `uv pip install -r requirements.txt`
- [ ] Create configuration system
- [ ] Implement basic logging framework

### Phase 2: Core Functionality
- [ ] Implement CSV processor with validation
- [ ] Create rate limiter with exponential backoff
- [ ] Implement Google Scholar searcher with rate limiting
- [ ] Add Sci-Hub integration
- [ ] Create download manager with file validation

### Phase 3: Integration and Testing
- [ ] Implement search strategy orchestrator
- [ ] Create main application entry point
- [ ] Add comprehensive error handling
- [ ] Implement progress tracking and reporting
- [ ] Create basic test suite

### Phase 4: Polish and Optimization
- [ ] Add command-line interface
- [ ] Implement resume functionality
- [ ] Optimize performance and memory usage
- [ ] Add comprehensive logging and reporting
- [ ] Create user documentation

## Technical Implementation Details

### Google Scholar Search Strategy
1. **Direct DOI Search**: `"10.1038/nature12345"` (highest success rate)
2. **Title + Author Search**: `"Paper Title" "First Author"` (fallback)
3. **Title Only Search**: `"Paper Title"` (last resort)

### Rate Limiting Implementation
```python
class RateLimiter:
    def __init__(self, requests_per_minute, delay_between_requests):
        self.requests_per_minute = requests_per_minute
        self.delay_between_requests = delay_between_requests
        self.last_request_time = 0
        self.request_count = 0
        self.reset_time = time.time() + 60
    
    def wait_if_needed(self):
        # Implementation for rate limiting
        pass
```

### Error Handling Strategy
- **Network Errors**: Retry with exponential backoff
- **Rate Limiting**: Implement delays and user agent rotation
- **Invalid DOIs**: Log and skip, continue processing
- **PDF Validation Failures**: Mark as failed with reason
- **Access Restrictions**: Try alternative sources

### File Validation
```python
def validate_pdf(file_path):
    """Validate downloaded PDF file"""
    try:
        with open(file_path, 'rb') as f:
            header = f.read(4)
            if header != b'%PDF':
                return False, "Invalid PDF header"
            
            # Check file size
            f.seek(0, 2)
            size = f.tell()
            if size < 1024:  # Less than 1KB
                return False, "File too small"
            
            return True, "Valid PDF"
    except Exception as e:
        return False, f"Validation error: {str(e)}"
```

## Performance Considerations

### Memory Management
- Process CSV in chunks for large files
- Stream downloads instead of loading entire files into memory
- Implement garbage collection for long-running processes

### Network Optimization
- Use connection pooling for HTTP requests
- Implement request timeouts to prevent hanging
- Cache successful search results

### Concurrency
- Consider async/await for I/O operations
- Implement thread pool for parallel downloads (with rate limiting)
- Use asyncio for non-blocking operations

## Testing Strategy

### Unit Tests
- CSV parsing and validation
- Rate limiting functionality
- File validation logic
- Error handling scenarios

### Integration Tests
- End-to-end download workflows
- Rate limiting behavior
- Error recovery mechanisms

### Performance Tests
- Large CSV file processing
- Memory usage optimization
- Network timeout handling

## Success Criteria

### Functional Requirements
- Success rate > 80% for papers with valid DOIs
- Average download time < 30 seconds per paper
- Zero data loss or corruption
- Comprehensive error reporting

### Technical Requirements
- Respect rate limits for all sources
- Handle network failures gracefully
- Provide detailed progress tracking
- Generate comprehensive logs

## Usage Example

```bash
# Setup environment
uv venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
uv pip install -r requirements.txt

# Run the application
python src/main.py --input papers.csv --output downloads/
```

This implementation plan focuses on practical, executable code with specific technical requirements and efficient strategies for avoiding rate limiting while maximizing download success rates. 