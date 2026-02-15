# Requirements Document: Kisan AI Sahayak

## Introduction

Kisan AI Sahayak is a voice-first, multimodal AI assistant designed to address the critical challenges faced by Indian smallholder farmers. The system provides crop disease detection, smart market intelligence, and personalized agricultural guidance through WhatsApp in local languages (Hindi/Marathi). By leveraging AWS AI services, the solution aims to reduce crop losses, increase farmer income, and provide timely expert advice to farmers who currently have limited access to agricultural extension services.

The system targets 146 million smallholder farmers in India who collectively lose ₹1.2 lakh crore annually due to late disease detection, poor market timing, and lack of personalized guidance. The MVP focuses on three major crops (Tomato, Wheat, Rice) and integrates with government e-NAM market data to provide actionable intelligence.

## Glossary

- **System**: The Kisan AI Sahayak platform including all AWS services, integrations, and WhatsApp interface
- **Farmer**: End user who interacts with the system via WhatsApp
- **Disease_Detection_Module**: Computer vision component that analyzes crop images and diagnoses diseases
- **Market_Intelligence_Module**: Component that analyzes mandi prices and provides selling recommendations
- **Conversation_Memory_Module**: Component that stores and retrieves farmer profiles, history, and context
- **WhatsApp_Interface**: Communication layer that handles message routing between Farmer and System
- **Transcription_Service**: Amazon Transcribe component that converts voice to text
- **AI_Engine**: Amazon Bedrock Claude Sonnet 4 that performs reasoning, analysis, and response generation
- **Voice_Synthesis_Service**: Amazon Polly component that converts text responses to voice
- **Database**: DynamoDB tables storing farmer data, conversation history, and crop information
- **e-NAM_API**: Government API providing live mandi prices for 247 commodities across 1,473 markets
- **Treatment_Advisory**: Recommendations for disease treatment including organic and chemical options
- **Price_Trend**: Historical price analysis showing 3-day moving average and predictions
- **Farmer_Profile**: Stored data including farmer location, crops grown, disease history, and preferences
- **Mandi**: Agricultural wholesale market where farmers sell produce
- **ONDC**: Open Network for Digital Commerce - digital marketplace alternative to physical mandis

## Requirements

### Requirement 1: Voice Input Processing

**User Story:** As a farmer, I want to send voice messages in my local language (Hindi/English), so that I can communicate naturally without typing.

#### Acceptance Criteria

1. WHEN a Farmer sends a voice message via WhatsApp, THE Transcription_Service SHALL convert it to text within 10 seconds
2. WHEN the voice message is in Hindi, THE Transcription_Service SHALL transcribe with minimum 85% accuracy
3. WHEN the voice message is in English, THE Transcription_Service SHALL transcribe with minimum 90% accuracy
4. WHEN transcription fails, THE System SHALL request the Farmer to resend the message or use text input
5. WHEN the voice message exceeds 60 seconds, THE System SHALL process it in segments and combine results
6. WHEN background noise is detected, THE System SHALL apply noise reduction before transcription

### Requirement 2: Crop Disease Detection

**User Story:** As a farmer, I want to send photos of my diseased crops and receive accurate diagnosis, so that I can treat the problem quickly and prevent crop loss.

#### Acceptance Criteria

1. WHEN a Farmer uploads a crop image, THE Disease_Detection_Module SHALL analyze it and return a diagnosis within 30 seconds
2. WHEN the image shows a recognizable disease, THE Disease_Detection_Module SHALL identify it with minimum 90% confidence
3. WHEN the disease is identified, THE System SHALL provide the disease name in Hindi and English
4. WHEN multiple diseases are detected in one image, THE System SHALL list all diseases with confidence scores
5. WHEN the image quality is too poor for analysis, THE System SHALL request a clearer photo with guidance on lighting and distance
6. WHEN the crop type is not supported (not Tomato, Wheat, or Rice), THE System SHALL inform the Farmer and suggest supported crops
7. WHEN no disease is detected, THE System SHALL confirm the crop appears healthy and suggest preventive measures

### Requirement 3: Treatment Advisory Generation

**User Story:** As a farmer, I want to receive treatment recommendations with cost analysis, so that I can choose the most suitable and affordable option.

#### Acceptance Criteria

1. WHEN a disease is diagnosed, THE System SHALL provide minimum 2 treatment options (organic and chemical)
2. WHEN providing treatment options, THE System SHALL include estimated cost per acre in rupees
3. WHEN providing treatment options, THE System SHALL include application method and dosage
4. WHEN providing treatment options, THE System SHALL consider current weather conditions for application suitability
5. WHEN the treatment requires specific weather conditions, THE System SHALL recommend optimal application timing
6. WHEN organic treatment is available, THE System SHALL prioritize it in recommendations
7. WHEN treatment requires safety precautions, THE System SHALL explicitly mention protective equipment needed

### Requirement 4: Market Price Intelligence

**User Story:** As a farmer, I want to know current mandi prices and selling recommendations, so that I can maximize my income by selling at the right time and place.

#### Acceptance Criteria

1. WHEN a Farmer requests market prices, THE Market_Intelligence_Module SHALL fetch live data from e-NAM_API within 5 seconds
2. WHEN displaying prices, THE System SHALL show the nearest 3 mandis based on Farmer location
3. WHEN displaying prices, THE System SHALL show current price, 3-day moving average, and trend direction
4. WHEN price trend is upward, THE System SHALL recommend waiting if crop condition allows
5. WHEN price trend is downward, THE System SHALL recommend immediate selling
6. WHEN a festival or seasonal event is approaching within 7 days, THE System SHALL factor it into price predictions
7. WHEN transport costs exceed 10% of crop value, THE System SHALL recommend local mandi over distant markets
8. WHEN ONDC digital marketplace offers better prices, THE System SHALL suggest it as an alternative

### Requirement 5: Conversational Memory and Context

**User Story:** As a farmer, I want the system to remember my previous interactions and crop history, so that I receive personalized advice without repeating information.

#### Acceptance Criteria

1. WHEN a Farmer first interacts with the System, THE Conversation_Memory_Module SHALL create a Farmer_Profile
2. WHEN a Farmer provides location information, THE Database SHALL store it in the Farmer_Profile
3. WHEN a disease is diagnosed, THE Database SHALL record it in the Farmer's disease history with timestamp
4. WHEN a treatment is recommended, THE Database SHALL track whether the Farmer applied it
5. WHEN a Farmer asks a follow-up question, THE System SHALL retrieve conversation context from the last 5 interactions
6. WHEN providing advice, THE AI_Engine SHALL reference the Farmer's crop history and past treatments
7. WHEN a recurring disease is detected, THE System SHALL alert the Farmer and suggest preventive measures

### Requirement 6: WhatsApp Interface Integration

**User Story:** As a farmer, I want to interact with the system through WhatsApp, so that I don't need to download a separate app or learn new technology.

#### Acceptance Criteria

1. WHEN a Farmer sends a message to the WhatsApp number, THE WhatsApp_Interface SHALL receive it within 2 seconds
2. WHEN the System generates a response, THE WhatsApp_Interface SHALL deliver it to the Farmer within 30 seconds total
3. WHEN a Farmer sends text, voice, or image, THE System SHALL accept all three input types
4. WHEN the System responds, THE WhatsApp_Interface SHALL support both text and voice output formats
5. WHEN multiple messages arrive simultaneously, THE System SHALL process them in order with queue management
6. WHEN the Farmer is offline, THE WhatsApp_Interface SHALL deliver responses when they reconnect
7. WHEN the System is unavailable, THE WhatsApp_Interface SHALL send an acknowledgment and retry message delivery

### Requirement 7: Voice Output Generation

**User Story:** As a farmer with limited literacy, I want to receive responses as voice messages in Hindi, so that I can understand the advice without reading.

#### Acceptance Criteria

1. WHEN the System generates a text response, THE Voice_Synthesis_Service SHALL convert it to Hindi voice
2. WHEN generating voice output, THE Voice_Synthesis_Service SHALL use natural prosody and appropriate speaking rate
3. WHEN the response contains technical terms, THE Voice_Synthesis_Service SHALL pronounce them clearly
4. WHEN the response exceeds 60 seconds of speech, THE System SHALL split it into multiple voice messages
5. WHEN the Farmer prefers text output, THE System SHALL remember this preference and skip voice synthesis
6. WHEN numbers or prices are included, THE Voice_Synthesis_Service SHALL speak them in Indian numbering format

### Requirement 8: Multi-Crop Support

**User Story:** As a farmer growing different crops, I want the system to handle Tomato, Wheat, and Rice, so that I can get advice for all my major crops.

#### Acceptance Criteria

1. WHEN a Farmer uploads an image, THE Disease_Detection_Module SHALL first identify the crop type
2. WHEN the crop is Tomato, THE Disease_Detection_Module SHALL detect diseases from a library of minimum 15 tomato diseases
3. WHEN the crop is Wheat, THE Disease_Detection_Module SHALL detect diseases from a library of minimum 10 wheat diseases
4. WHEN the crop is Rice, THE Disease_Detection_Module SHALL detect diseases from a library of minimum 12 rice diseases
5. WHEN the crop type cannot be determined, THE System SHALL ask the Farmer to specify the crop
6. WHEN providing treatment advice, THE System SHALL use crop-specific dosage and application methods

### Requirement 9: Performance and Scalability

**User Story:** As a farmer in a rural area with slow internet, I want the system to respond quickly even during peak hours, so that I can get timely advice.

#### Acceptance Criteria

1. WHEN processing a request, THE System SHALL complete end-to-end response within 30 seconds
2. WHEN 100 concurrent Farmers are using the System, THE System SHALL maintain response time under 30 seconds
3. WHEN the e-NAM_API is slow, THE System SHALL use cached price data from the last 2 hours
4. WHEN AWS services experience latency, THE System SHALL implement exponential backoff with maximum 3 retries
5. WHEN image upload fails due to network issues, THE System SHALL support resumable uploads
6. WHILE the System is under load, THE System SHALL prioritize disease detection requests over market queries

### Requirement 10: Cost Optimization

**User Story:** As a system operator, I want to minimize AWS costs, so that the service remains affordable and sustainable at scale.

#### Acceptance Criteria

1. WHEN processing requests, THE System SHALL maintain average cost below ₹5 per Farmer per month
2. WHEN storing conversation history, THE Database SHALL implement automatic archival of data older than 90 days
3. WHEN generating voice output, THE System SHALL cache common responses to reduce Polly API calls
4. WHEN analyzing images, THE System SHALL compress images to maximum 1MB before sending to AI_Engine
5. WHEN the Farmer is inactive for 30 days, THE System SHALL move their data to cold storage
6. WHEN using Bedrock, THE System SHALL use the most cost-effective model that meets accuracy requirements

### Requirement 11: Security and Privacy

**User Story:** As a farmer, I want my personal information and crop data to be secure, so that my privacy is protected.

#### Acceptance Criteria

1. WHEN storing Farmer data, THE Database SHALL encrypt all data at rest using AES-256
2. WHEN transmitting data between services, THE System SHALL use TLS 1.2 or higher
3. WHEN a Farmer requests data deletion, THE System SHALL remove all personal data within 30 days
4. WHEN accessing the Database, THE System SHALL use IAM roles with least privilege access
5. WHEN logging interactions, THE System SHALL not log personally identifiable information in plain text
6. WHEN a Farmer shares crop images, THE System SHALL not use them for purposes beyond diagnosis without explicit consent
7. IF unauthorized access is detected, THEN THE System SHALL lock the affected account and alert administrators

### Requirement 12: Error Handling and Fallbacks

**User Story:** As a farmer, I want the system to handle errors gracefully, so that I always receive helpful responses even when something goes wrong.

#### Acceptance Criteria

1. WHEN the AI_Engine fails to generate a response, THE System SHALL provide a fallback message and suggest contacting support
2. WHEN the e-NAM_API is unavailable, THE System SHALL use cached price data and inform the Farmer of the data age
3. WHEN image analysis fails, THE System SHALL request a new image with specific guidance on quality requirements
4. WHEN transcription confidence is below 70%, THE System SHALL ask the Farmer to confirm the transcribed text
5. WHEN the Database is unavailable, THE System SHALL process the request without memory context and log for retry
6. WHEN external API rate limits are exceeded, THE System SHALL queue requests and inform the Farmer of expected delay
7. IF any critical service fails, THEN THE System SHALL send an SMS fallback with helpline number

### Requirement 13: Localization and Language Support

**User Story:** As a Hindi-speaking farmer, I want all interactions in my language with culturally appropriate content, so that I can understand and trust the advice.

#### Acceptance Criteria

1. WHEN the System generates responses, THE AI_Engine SHALL use Hindi (Devanagari script) as the primary language
2. WHEN providing measurements, THE System SHALL use Indian units (acre, quintal, kg) not metric tons or hectares
3. WHEN mentioning prices, THE System SHALL use Indian numbering format (lakh, crore) and ₹ symbol
4. WHEN suggesting timing, THE System SHALL reference Indian agricultural calendar and festivals
5. WHEN the Farmer uses English words in Hindi conversation, THE System SHALL understand code-mixing
6. WHEN providing disease names, THE System SHALL include both scientific name and local Hindi name
7. WHERE the Farmer prefers Marathi, THE System SHALL support Marathi language for voice input and output

### Requirement 14: Weather Integration

**User Story:** As a farmer, I want treatment recommendations that consider current and forecasted weather, so that I apply treatments at the right time for maximum effectiveness.

#### Acceptance Criteria

1. WHEN providing treatment advice, THE System SHALL fetch 7-day weather forecast for the Farmer's location
2. WHEN rain is forecasted within 24 hours, THE System SHALL warn against applying water-soluble treatments
3. WHEN temperature exceeds 35°C, THE System SHALL recommend early morning or evening application
4. WHEN high winds are forecasted, THE System SHALL warn against spray applications
5. WHEN humidity is below 40%, THE System SHALL recommend additional watering after treatment
6. WHEN the weather API is unavailable, THE System SHALL provide treatment advice with a disclaimer about weather conditions

### Requirement 15: Onboarding and User Guidance

**User Story:** As a new farmer user, I want clear instructions on how to use the system, so that I can quickly start getting value from it.

#### Acceptance Criteria

1. WHEN a Farmer first contacts the System, THE System SHALL send a welcome message in Hindi explaining capabilities
2. WHEN the welcome message is sent, THE System SHALL include example queries the Farmer can try
3. WHEN a Farmer sends an unclear message, THE System SHALL provide guidance on how to phrase questions
4. WHEN a Farmer uploads a poor-quality image, THE System SHALL provide specific tips on taking better crop photos
5. WHEN a Farmer asks "what can you do", THE System SHALL list all three modules with examples
6. WHEN the Farmer completes their first successful interaction, THE System SHALL encourage them to save the WhatsApp number

### Requirement 16: Monitoring and Analytics

**User Story:** As a system operator, I want to track usage patterns and system health, so that I can optimize performance and identify issues proactively.

#### Acceptance Criteria

1. WHEN a request is processed, THE System SHALL log response time, service used, and success/failure status
2. WHEN an error occurs, THE System SHALL log error type, service affected, and context for debugging
3. WHEN daily usage exceeds 1000 requests, THE System SHALL send an alert to administrators
4. WHEN average response time exceeds 40 seconds, THE System SHALL trigger performance investigation
5. WHEN disease detection accuracy drops below 85%, THE System SHALL flag for model retraining
6. WHEN a Farmer provides feedback, THE Database SHALL store it with associated interaction ID for analysis

### Requirement 17: Treatment Outcome Tracking

**User Story:** As a farmer, I want to report treatment results, so that the system can learn and improve recommendations over time.

#### Acceptance Criteria

1. WHEN a treatment is recommended, THE System SHALL follow up after 7 days to ask about effectiveness
2. WHEN a Farmer reports treatment success, THE Database SHALL record it with disease and treatment details
3. WHEN a Farmer reports treatment failure, THE System SHALL ask for updated crop photos and provide alternative treatment
4. WHEN treatment outcome data is collected, THE System SHALL use it to improve future recommendations
5. WHEN a treatment consistently shows poor results, THE System SHALL deprioritize it in future recommendations

### Requirement 18: Market Linkage and ONDC Integration

**User Story:** As a farmer, I want to know about digital marketplace options, so that I can sell directly to buyers and get better prices.

#### Acceptance Criteria

1. WHEN displaying market prices, THE System SHALL compare physical mandi prices with ONDC digital marketplace prices
2. WHEN ONDC offers 15% or higher price premium, THE System SHALL recommend digital selling
3. WHEN recommending ONDC, THE System SHALL explain the process and required documentation
4. WHEN the Farmer is interested in ONDC, THE System SHALL provide contact information for onboarding support
5. WHEN the crop quality is premium (based on disease history), THE System SHALL highlight ONDC as preferred option

### Requirement 19: Offline Capability and SMS Fallback

**User Story:** As a farmer in an area with intermittent internet, I want basic functionality even when WhatsApp is unavailable, so that I can still get critical information.

#### Acceptance Criteria

1. WHEN WhatsApp is unavailable for more than 5 minutes, THE System SHALL send SMS with helpline number
2. WHEN a Farmer sends SMS to the helpline, THE System SHALL respond with basic price information via SMS
3. WHEN internet connectivity is restored, THE System SHALL sync any missed interactions
4. WHEN the Farmer has pending responses, THE System SHALL deliver them in order when they reconnect

### Requirement 20: Data Export and Portability

**User Story:** As a farmer, I want to export my crop history and treatment records, so that I can share them with agricultural officers or use them for loan applications.

#### Acceptance Criteria

1. WHEN a Farmer requests data export, THE System SHALL generate a PDF report with all crop history
2. WHEN generating the report, THE System SHALL include disease timeline, treatments applied, and outcomes
3. WHEN generating the report, THE System SHALL include market transactions and price history
4. WHEN the report is ready, THE System SHALL send it via WhatsApp as a downloadable file
5. WHEN the Farmer requests data deletion, THE System SHALL provide the export before deletion
