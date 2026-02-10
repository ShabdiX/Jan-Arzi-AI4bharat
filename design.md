# Design Document: Jan-Arzi

## Overview

Jan-Arzi is a voice-to-formal-document bridge designed to empower rural Indian villagers to create formal government applications without literacy requirements or paid agents. The system follows a three-stage pipeline: (1) Voice capture and transcription, (2) Text formalization using LLM, and (3) PDF generation and delivery via WhatsApp.

### Key Design Principles

1. **Simplicity First**: Mobile-first UI with minimal text, large buttons, and visual feedback for low-literacy users
2. **Resilience**: Robust error handling and retry mechanisms for unreliable network conditions
3. **Privacy**: Secure data handling with encryption and automatic cleanup
4. **Accessibility**: Support for multiple Indian languages and dialects
5. **Real-world Impact**: Optimized for the complete user journey from voice input to printed document submission

### Technology Stack

**Frontend:**
- Progressive Web App (PWA) for mobile-first experience
- React or Vue.js for UI components
- Web Audio API for voice recording
- Responsive design with large touch targets

**Backend:**
- Python (FastAPI or Flask) for API server
- Celery for asynchronous task processing
- Redis for task queue and caching

**AI Services:**
- OpenAI Whisper or Bhashini API for speech-to-text transcription
- OpenAI GPT-4 or similar LLM for text formalization
- Language detection for multi-dialect support

**Storage & Infrastructure:**
- Cloud storage (AWS S3 or similar) for temporary audio files
- PostgreSQL for metadata and request tracking
- WhatsApp Business API for PDF delivery

**PDF Generation:**
- ReportLab (Python) or PDFKit for document generation
- Custom templates for official formatting

## Architecture

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        User (Mobile)                         │
│                     ┌──────────────┐                         │
│                     │  PWA Frontend │                        │
│                     │  (React/Vue)  │                        │
│                     └───────┬──────┘                         │
└─────────────────────────────┼───────────────────────────────┘
                              │ HTTPS
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      API Gateway Layer                       │
│                     ┌──────────────┐                         │
│                     │  FastAPI/Flask│                        │
│                     │  REST API     │                        │
│                     └───────┬──────┘                         │
└─────────────────────────────┼───────────────────────────────┘
                              │
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼
┌──────────────────┐ ┌──────────────┐ ┌──────────────┐
│  Audio Upload    │ │  Status      │ │  Download    │
│  Endpoint        │ │  Endpoint    │ │  Endpoint    │
└────────┬─────────┘ └──────────────┘ └──────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│                    Processing Pipeline                       │
│                                                               │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │  Stage 1 │───▶│  Stage 2 │───▶│  Stage 3 │              │
│  │Transcribe│    │Formalize │    │Generate  │              │
│  │  Audio   │    │   Text   │    │   PDF    │              │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘              │
│       │               │               │                      │
│       ▼               ▼               ▼                      │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                │
│  │ Whisper │    │  GPT-4  │    │ReportLab│                │
│  │Bhashini │    │   LLM   │    │ PDFKit  │                │
│  └─────────┘    └─────────┘    └─────────┘                │
│                                                               │
│              Managed by Celery Task Queue                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Delivery Layer                          │
│                                                               │
│  ┌──────────────────┐         ┌──────────────────┐         │
│  │  WhatsApp API    │         │  Direct Download │         │
│  │  (Primary)       │         │  (Fallback)      │         │
│  └──────────────────┘         └──────────────────┘         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Storage Layer                           │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Cloud       │  │  PostgreSQL  │  │    Redis     │     │
│  │  Storage     │  │  (Metadata)  │  │  (Cache)     │     │
│  │  (Audio/PDF) │  │              │  │              │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### Processing Flow

1. **Voice Capture**: User records audio via PWA frontend
2. **Upload**: Audio file uploaded to API server, stored in cloud storage
3. **Async Processing**: Celery task initiated for three-stage pipeline
4. **Stage 1 - Transcription**: Audio sent to Whisper/Bhashini API, text returned
5. **Stage 2 - Formalization**: Transcribed text sent to LLM with prompt template, formal application returned
6. **Stage 3 - PDF Generation**: Formal text rendered into PDF with official formatting
7. **Delivery**: PDF sent via WhatsApp API and/or made available for direct download
8. **Cleanup**: Audio and PDF files deleted after retention period (24-48 hours)

### Component Interaction

- **Frontend ↔ API**: RESTful JSON communication over HTTPS
- **API ↔ Task Queue**: Celery tasks for async processing
- **Task Queue ↔ AI Services**: HTTP API calls to Whisper, LLM services
- **Task Queue ↔ Storage**: File operations for audio/PDF storage
- **API ↔ WhatsApp**: WhatsApp Business API for message delivery

## Components and Interfaces

### Frontend Components

#### 1. VoiceRecorder Component
**Responsibility**: Capture audio input from user

**Interface:**
```typescript
interface VoiceRecorderProps {
  onRecordingComplete: (audioBlob: Blob) => void;
  onError: (error: Error) => void;
  maxDuration: number; // seconds
}

interface VoiceRecorderState {
  isRecording: boolean;
  duration: number;
  audioLevel: number; // for visual feedback
}
```

**Key Methods:**
- `startRecording()`: Initialize MediaRecorder and begin capture
- `stopRecording()`: Finalize recording and return Blob
- `cancelRecording()`: Discard current recording

#### 2. ApplicationPreview Component
**Responsibility**: Display formatted application text before PDF generation

**Interface:**
```typescript
interface ApplicationPreviewProps {
  formalText: FormalApplication;
  onApprove: () => void;
  onRegenerate: () => void;
}

interface FormalApplication {
  subject: string;
  salutation: string;
  body: string;
  closing: string;
  date: string;
}
```

#### 3. StatusIndicator Component
**Responsibility**: Show processing progress with visual feedback

**Interface:**
```typescript
interface StatusIndicatorProps {
  stage: 'recording' | 'uploading' | 'transcribing' | 'formalizing' | 'generating' | 'complete' | 'error';
  progress: number; // 0-100
  message: string;
}
```

#### 4. DeliveryOptions Component
**Responsibility**: Collect phone number and manage PDF delivery

**Interface:**
```typescript
interface DeliveryOptionsProps {
  pdfUrl: string;
  onWhatsAppSend: (phoneNumber: string) => void;
  onDirectDownload: () => void;
}
```

### Backend API Endpoints

#### POST /api/v1/applications
**Purpose**: Upload audio and initiate processing

**Request:**
```json
{
  "audio": "base64_encoded_audio_data",
  "format": "webm|mp3|wav",
  "language_hint": "hi|bho|en-IN",
  "phone_number": "+91XXXXXXXXXX" // optional
}
```

**Response:**
```json
{
  "application_id": "uuid",
  "status": "processing",
  "estimated_time": 45 // seconds
}
```

**Status Codes:**
- 202: Accepted, processing started
- 400: Invalid request (bad audio format, missing data)
- 413: Audio file too large
- 500: Server error

#### GET /api/v1/applications/{application_id}
**Purpose**: Check processing status

**Response:**
```json
{
  "application_id": "uuid",
  "status": "transcribing|formalizing|generating|complete|failed",
  "progress": 65,
  "transcribed_text": "...", // available after transcription
  "formal_application": { // available after formalization
    "subject": "...",
    "salutation": "...",
    "body": "...",
    "closing": "..."
  },
  "pdf_url": "https://...", // available when complete
  "error": "..." // if status is failed
}
```

**Status Codes:**
- 200: Success
- 404: Application ID not found
- 500: Server error

#### POST /api/v1/applications/{application_id}/regenerate
**Purpose**: Regenerate formal text with additional context

**Request:**
```json
{
  "additional_context": "Please mention electricity department specifically",
  "department": "Electricity Board"
}
```

**Response:**
```json
{
  "formal_application": {
    "subject": "...",
    "salutation": "...",
    "body": "...",
    "closing": "..."
  }
}
```

#### POST /api/v1/applications/{application_id}/deliver
**Purpose**: Send PDF via WhatsApp

**Request:**
```json
{
  "phone_number": "+91XXXXXXXXXX",
  "delivery_method": "whatsapp"
}
```

**Response:**
```json
{
  "delivery_status": "sent|failed",
  "message": "PDF sent successfully to WhatsApp"
}
```

**Status Codes:**
- 200: Delivery successful
- 400: Invalid phone number
- 404: Application not found or PDF not ready
- 500: Delivery service error

### Backend Service Components

#### 1. TranscriptionService
**Responsibility**: Convert audio to text using AI services

**Interface:**
```python
class TranscriptionService:
    def transcribe(
        self,
        audio_file_path: str,
        language_hint: str = "hi"
    ) -> TranscriptionResult:
        """
        Transcribe audio file to text.
        
        Args:
            audio_file_path: Path to audio file in storage
            language_hint: ISO language code hint
            
        Returns:
            TranscriptionResult with text and confidence
            
        Raises:
            TranscriptionError: If transcription fails
        """
        pass

@dataclass
class TranscriptionResult:
    text: str
    language_detected: str
    confidence: float
    duration: float
```

**Implementation Notes:**
- Use OpenAI Whisper API or Bhashini API
- Implement retry logic for API failures
- Support multiple audio formats (webm, mp3, wav)
- Handle language detection for multi-dialect support

#### 2. FormalizationService
**Responsibility**: Convert informal text to formal application format

**Interface:**
```python
class FormalizationService:
    def formalize(
        self,
        informal_text: str,
        department: Optional[str] = None,
        additional_context: Optional[str] = None
    ) -> FormalApplication:
        """
        Convert informal text to formal application.
        
        Args:
            informal_text: Transcribed user speech
            department: Target government department
            additional_context: Extra instructions for LLM
            
        Returns:
            FormalApplication with structured fields
            
        Raises:
            FormalizationError: If LLM processing fails
        """
        pass

@dataclass
class FormalApplication:
    subject: str
    salutation: str
    body: str
    closing: str
    date: str
    department: Optional[str]
```

**LLM Prompt Template:**
```
You are an expert in drafting formal government applications in India.

Convert the following informal complaint into a formal application letter suitable for submission to a government department.

Informal text: "{informal_text}"
{department_context}

Generate a formal application with:
1. Subject line (concise, specific)
2. Salutation (appropriate for government official)
3. Body (clear problem description with relevant details)
4. Closing (respectful, with space for signature)

Use formal Hindi or English as appropriate. Maintain all factual details from the original text.

Output format:
Subject: [subject line]
Salutation: [greeting]
Body: [main content]
Closing: [respectful closing]
```

#### 3. PDFGeneratorService
**Responsibility**: Create formatted PDF documents

**Interface:**
```python
class PDFGeneratorService:
    def generate_pdf(
        self,
        formal_application: FormalApplication,
        output_path: str
    ) -> str:
        """
        Generate PDF from formal application.
        
        Args:
            formal_application: Structured application data
            output_path: Where to save PDF file
            
        Returns:
            Path to generated PDF file
            
        Raises:
            PDFGenerationError: If PDF creation fails
        """
        pass
```

**PDF Layout Specifications:**
- Page size: A4 (210mm × 297mm)
- Margins: Top 25mm, Bottom 25mm, Left 20mm, Right 20mm
- Font: Noto Sans (supports Devanagari) or Arial
- Font sizes: Subject 14pt bold, Body 12pt, Date/Signature 11pt
- Line spacing: 1.5
- Signature space: 40mm from bottom
- Date field: Top right corner

#### 4. WhatsAppDeliveryService
**Responsibility**: Send PDF via WhatsApp Business API

**Interface:**
```python
class WhatsAppDeliveryService:
    def send_pdf(
        self,
        phone_number: str,
        pdf_path: str,
        message: str
    ) -> DeliveryResult:
        """
        Send PDF document via WhatsApp.
        
        Args:
            phone_number: Recipient phone with country code
            pdf_path: Path to PDF file
            message: Accompanying text message
            
        Returns:
            DeliveryResult with status
            
        Raises:
            DeliveryError: If sending fails
        """
        pass

@dataclass
class DeliveryResult:
    success: bool
    message_id: Optional[str]
    error: Optional[str]
```

**WhatsApp Message Template:**
```
नमस्ते! आपका आवेदन पत्र तैयार है।

Hello! Your application letter is ready.

इस PDF को डाउनलोड करें, प्रिंट करें और संबंधित विभाग में जमा करें।
Download this PDF, print it, and submit to the relevant department.

- Jan-Arzi Team
```

#### 5. StorageService
**Responsibility**: Manage file storage and cleanup

**Interface:**
```python
class StorageService:
    def upload_audio(
        self,
        audio_data: bytes,
        application_id: str,
        format: str
    ) -> str:
        """Upload audio file and return storage path."""
        pass
    
    def upload_pdf(
        self,
        pdf_path: str,
        application_id: str
    ) -> str:
        """Upload PDF and return public URL."""
        pass
    
    def delete_files(
        self,
        application_id: str
    ) -> None:
        """Delete all files for an application."""
        pass
    
    def schedule_cleanup(
        self,
        application_id: str,
        delay_hours: int = 48
    ) -> None:
        """Schedule automatic file deletion."""
        pass
```

### Celery Task Pipeline

#### Task: process_application
**Purpose**: Orchestrate the complete processing pipeline

```python
@celery.task(bind=True)
def process_application(
    self,
    application_id: str,
    audio_path: str,
    language_hint: str,
    phone_number: Optional[str]
) -> None:
    """
    Process application through complete pipeline.
    
    Updates application status at each stage.
    Handles errors and retries.
    """
    try:
        # Stage 1: Transcription
        self.update_state(state='TRANSCRIBING', meta={'progress': 20})
        transcription = transcription_service.transcribe(audio_path, language_hint)
        
        # Stage 2: Formalization
        self.update_state(state='FORMALIZING', meta={'progress': 50})
        formal_app = formalization_service.formalize(transcription.text)
        
        # Stage 3: PDF Generation
        self.update_state(state='GENERATING', meta={'progress': 75})
        pdf_path = pdf_generator.generate_pdf(formal_app, f"/tmp/{application_id}.pdf")
        pdf_url = storage_service.upload_pdf(pdf_path, application_id)
        
        # Stage 4: Delivery (if phone provided)
        if phone_number:
            self.update_state(state='DELIVERING', meta={'progress': 90})
            whatsapp_service.send_pdf(phone_number, pdf_path, DEFAULT_MESSAGE)
        
        # Complete
        self.update_state(state='COMPLETE', meta={'progress': 100, 'pdf_url': pdf_url})
        
        # Schedule cleanup
        storage_service.schedule_cleanup(application_id, delay_hours=48)
        
    except Exception as e:
        self.update_state(state='FAILED', meta={'error': str(e)})
        raise
```

## Data Models

### Application Model
**Purpose**: Track application processing state

```python
class Application(Base):
    __tablename__ = 'applications'
    
    id = Column(UUID, primary_key=True, default=uuid4)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, onupdate=datetime.utcnow)
    
    # Input data
    audio_path = Column(String, nullable=False)
    audio_format = Column(String, nullable=False)
    language_hint = Column(String, default='hi')
    phone_number = Column(String, nullable=True)  # Encrypted
    
    # Processing state
    status = Column(Enum(ApplicationStatus), default=ApplicationStatus.PENDING)
    progress = Column(Integer, default=0)
    error_message = Column(Text, nullable=True)
    
    # Results
    transcribed_text = Column(Text, nullable=True)
    formal_subject = Column(String, nullable=True)
    formal_salutation = Column(Text, nullable=True)
    formal_body = Column(Text, nullable=True)
    formal_closing = Column(Text, nullable=True)
    pdf_url = Column(String, nullable=True)
    
    # Delivery
    whatsapp_sent = Column(Boolean, default=False)
    whatsapp_message_id = Column(String, nullable=True)
    
    # Cleanup
    files_deleted = Column(Boolean, default=False)
    deletion_scheduled_at = Column(DateTime, nullable=True)

class ApplicationStatus(enum.Enum):
    PENDING = "pending"
    TRANSCRIBING = "transcribing"
    FORMALIZING = "formalizing"
    GENERATING = "generating"
    DELIVERING = "delivering"
    COMPLETE = "complete"
    FAILED = "failed"
```

### Configuration Model
**Purpose**: Store system configuration

```python
class SystemConfig(Base):
    __tablename__ = 'system_config'
    
    key = Column(String, primary_key=True)
    value = Column(JSON, nullable=False)
    updated_at = Column(DateTime, onupdate=datetime.utcnow)

# Example configurations:
# - max_audio_duration: 300 (seconds)
# - max_audio_size: 10485760 (10MB in bytes)
# - file_retention_hours: 48
# - supported_languages: ["hi", "bho", "en-IN"]
# - whatsapp_enabled: true
```

### Analytics Model (Optional)
**Purpose**: Track usage metrics for impact measurement

```python
class UsageMetrics(Base):
    __tablename__ = 'usage_metrics'
    
    id = Column(UUID, primary_key=True, default=uuid4)
    timestamp = Column(DateTime, default=datetime.utcnow)
    
    application_id = Column(UUID, ForeignKey('applications.id'))
    language_detected = Column(String)
    audio_duration = Column(Float)
    processing_time = Column(Float)
    success = Column(Boolean)
    delivery_method = Column(String)  # whatsapp, download, none
```

