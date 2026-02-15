# Design Document: Kisan AI Sahayak

## Overview

Kisan AI Sahayak is a serverless, event-driven AI system built on AWS that provides voice-first agricultural assistance to smallholder farmers through WhatsApp. The architecture leverages five core AWS services orchestrated through Lambda functions to deliver multimodal AI capabilities including crop disease detection, market intelligence, and conversational memory.

### Design Philosophy

1. **Voice-First**: Prioritize voice interaction over text to accommodate low-literacy users
2. **Serverless**: Use managed AWS services to minimize operational overhead and optimize costs
3. **Modular**: Separate concerns into distinct modules (disease detection, market intelligence, memory) for independent scaling
4. **Resilient**: Implement fallbacks at every layer to ensure farmers always receive responses
5. **Cost-Conscious**: Target <₹5/farmer/month through caching, compression, and intelligent service selection

### Key Design Decisions

**Why WhatsApp over Native App**: 487M WhatsApp users in India with 75% rural penetration vs app download friction
**Why Bedrock Claude Sonnet 4**: Multimodal capabilities (image + text) with superior reasoning for complex agricultural advice
**Why DynamoDB over RDS**: Single-digit millisecond latency, automatic scaling, and pay-per-request pricing fits usage patterns
**Why Twilio**: Production-grade WhatsApp Business API with webhook support and 99.95% uptime SLA


## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Farmer (WhatsApp)                        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Twilio WhatsApp Business API                  │
│                         (Webhook Handler)                        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API Gateway (REST API)                      │
│                    /webhook/whatsapp (POST)                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Lambda: Orchestrator Function                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 1. Parse incoming message (text/voice/image)            │   │
│  │ 2. Retrieve farmer context from DynamoDB                │   │
│  │ 3. Route to appropriate module                          │   │
│  │ 4. Invoke AI Engine for response generation             │   │
│  │ 5. Store interaction in DynamoDB                        │   │
│  │ 6. Return response to Twilio                            │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────┬────────┬────────┬────────┬────────┬────────┬──────────────┘
      │        │        │        │        │        │
      ▼        ▼        ▼        ▼        ▼        ▼
┌─────────┐ ┌──────┐ ┌──────┐ ┌────────┐ ┌──────┐ ┌────────────┐
│Transcribe│ │Bedrock│ │Polly │ │DynamoDB│ │S3    │ │External APIs│
│         │ │Claude │ │      │ │        │ │      │ │            │
│Voice→Text│ │Sonnet4│ │Text→ │ │Farmer  │ │Image │ │• e-NAM     │
│         │ │       │ │Voice │ │Profiles│ │Store │ │• Weather   │
│         │ │Image  │ │      │ │History │ │      │ │            │
│         │ │Analysis│ │      │ │        │ │      │ │            │
└─────────┘ └──────┘ └──────┘ └────────┘ └──────┘ └────────────┘
```

### Request Flow Patterns

**Pattern 1: Voice Query with Disease Detection**
```
Farmer sends voice + image → Twilio → API Gateway → Lambda Orchestrator
  ↓
  1. Upload image to S3 (generate presigned URL)
  2. Invoke Transcribe on voice audio
  3. Retrieve farmer context from DynamoDB
  4. Invoke Bedrock with: image URL + transcribed text + farmer context
  5. Bedrock analyzes image, diagnoses disease, generates treatment advice
  6. Store diagnosis in DynamoDB (farmer history)
  7. Invoke Polly to convert response to Hindi voice
  8. Return voice message to Twilio → Farmer
```

**Pattern 2: Market Price Query**
```
Farmer asks "aaj ke bhav kya hain?" → Transcribe → Lambda
  ↓
  1. Retrieve farmer location from DynamoDB
  2. Call e-NAM API for nearest 3 mandis
  3. Calculate 3-day moving average from cached data
  4. Invoke Bedrock with: prices + weather + farmer crop history
  5. Bedrock generates contextual recommendation
  6. Polly converts to voice
  7. Return to Farmer
```

### AWS Service Configuration

**API Gateway**
- Type: REST API (not HTTP API) for request validation
- Throttling: 100 requests/second burst, 50 steady state
- Timeout: 29 seconds (Lambda max - 1 second buffer)
- CORS: Disabled (webhook only, no browser access)

**Lambda Orchestrator**
- Runtime: Python 3.11
- Memory: 1024 MB (balance between cost and performance)
- Timeout: 30 seconds
- Concurrency: Reserved 50, Provisioned 5 (for cold start mitigation)
- Environment Variables: BEDROCK_MODEL_ID, DYNAMODB_TABLE, S3_BUCKET, TWILIO_AUTH_TOKEN

**Amazon Transcribe**
- Language: hi-IN (Hindi), en-IN (English)
- Model: Standard (not medical/call analytics)
- Output: JSON with confidence scores
- Custom Vocabulary: Agricultural terms (mandi, quintal, kharif, rabi)

**Amazon Bedrock**
- Model: anthropic.claude-sonnet-4-20250514
- Max Tokens: 2048
- Temperature: 0.3 (lower for factual accuracy)
- System Prompt: Detailed agricultural expert persona (see Prompt Engineering section)

**Amazon Polly**
- Voice: Aditi (Hindi, female, neural engine)
- Output Format: MP3 (compressed for WhatsApp)
- Speech Rate: 90% (slightly slower for clarity)
- Prosody: Conversational style

**DynamoDB**
- Billing Mode: Pay-per-request (on-demand)
- Tables: FarmerProfiles, ConversationHistory, DiseaseRecords, PriceCache
- Encryption: AWS managed keys (default)
- TTL: Enabled on ConversationHistory (90 days)

**S3**
- Bucket: kisan-ai-crop-images-{region}-{account-id}
- Lifecycle: Delete objects after 30 days
- Encryption: SSE-S3
- Access: Private (presigned URLs only)


## Components and Interfaces

### Component 1: Message Router

**Responsibility**: Parse incoming WhatsApp messages and route to appropriate handler

**Interface**:
```python
class MessageRouter:
    def route_message(webhook_payload: dict) -> MessageContext:
        """
        Parse Twilio webhook payload and extract message components
        
        Args:
            webhook_payload: Raw POST data from Twilio
            
        Returns:
            MessageContext with:
              - farmer_phone: str (E.164 format)
              - message_type: Enum[TEXT, VOICE, IMAGE, LOCATION]
              - content: Union[str, bytes, tuple[float, float]]
              - media_url: Optional[str]
              - timestamp: datetime
        """
```

**Implementation Notes**:
- Twilio sends different payload structures for text vs media messages
- Voice messages arrive as audio/ogg files (need conversion to formats Transcribe accepts)
- Images arrive as JPEG/PNG with MediaUrl field
- Must validate Twilio signature to prevent spoofing (X-Twilio-Signature header)

### Component 2: Farmer Context Manager

**Responsibility**: Retrieve and update farmer profile and conversation history

**Interface**:
```python
class FarmerContextManager:
    def get_farmer_profile(phone: str) -> FarmerProfile:
        """
        Retrieve farmer profile from DynamoDB
        
        Returns:
            FarmerProfile:
              - phone: str (primary key)
              - name: Optional[str]
              - location: tuple[float, float] (lat, lon)
              - preferred_language: str (default: "hi")
              - crops: list[str]
              - created_at: datetime
              - last_interaction: datetime
        """
    
    def get_conversation_history(phone: str, limit: int = 5) -> list[Interaction]:
        """
        Retrieve recent conversation history
        
        Returns list of Interaction objects with:
          - interaction_id: str
          - timestamp: datetime
          - user_message: str
          - system_response: str
          - module_used: str (disease_detection, market_intel, general)
        """
    
    def update_profile(phone: str, updates: dict) -> None:
        """Update farmer profile with new information"""
    
    def store_interaction(phone: str, interaction: Interaction) -> None:
        """Store new interaction in conversation history"""
```

**DynamoDB Schema**:

**Table: FarmerProfiles**
```
Primary Key: phone (String)
Attributes:
  - name (String)
  - location (Map: {lat: Number, lon: Number})
  - preferred_language (String)
  - crops (List[String])
  - created_at (Number - Unix timestamp)
  - last_interaction (Number - Unix timestamp)
  - total_queries (Number)
  - premium_status (Boolean)

GSI: location-index
  - Partition Key: location_grid (String - geohash)
  - Sort Key: last_interaction
```

**Table: ConversationHistory**
```
Primary Key: phone (String)
Sort Key: timestamp (Number - Unix timestamp)
Attributes:
  - interaction_id (String - UUID)
  - user_message (String)
  - system_response (String)
  - module_used (String)
  - media_urls (List[String])
  - ttl (Number - for automatic deletion after 90 days)

GSI: interaction-type-index
  - Partition Key: module_used
  - Sort Key: timestamp
```

**Table: DiseaseRecords**
```
Primary Key: phone (String)
Sort Key: diagnosis_date (Number - Unix timestamp)
Attributes:
  - disease_id (String)
  - crop_type (String)
  - disease_name (String)
  - confidence_score (Number)
  - treatment_recommended (String)
  - treatment_applied (Boolean)
  - outcome (String - success/failure/pending)
  - image_url (String)
  - follow_up_date (Number)
```

**Table: PriceCache**
```
Primary Key: commodity_mandi (String - "tomato_delhi")
Sort Key: date (String - YYYY-MM-DD)
Attributes:
  - price (Number)
  - unit (String)
  - volume (Number)
  - trend (String - up/down/stable)
  - ttl (Number - 7 days)
```

### Component 3: Disease Detection Module

**Responsibility**: Analyze crop images and diagnose diseases using Bedrock

**Interface**:
```python
class DiseaseDetectionModule:
    def analyze_crop_image(
        image_url: str,
        farmer_context: FarmerProfile,
        voice_description: Optional[str]
    ) -> DiagnosisResult:
        """
        Analyze crop image for disease detection
        
        Args:
            image_url: S3 presigned URL or public URL
            farmer_context: Farmer profile for personalization
            voice_description: Optional transcribed voice description
            
        Returns:
            DiagnosisResult:
              - crop_type: str
              - diseases: list[Disease]
              - confidence: float
              - image_quality_score: float
              - requires_better_image: bool
        """
    
    def generate_treatment_plan(
        diagnosis: DiagnosisResult,
        weather: WeatherForecast,
        farmer_history: list[DiseaseRecord]
    ) -> TreatmentPlan:
        """
        Generate treatment recommendations based on diagnosis
        
        Returns:
            TreatmentPlan:
              - treatments: list[Treatment]
              - application_timing: str
              - cost_estimate: dict[str, float]
              - preventive_measures: list[str]
        """
```

**Bedrock Prompt Structure for Disease Detection**:
```
System Prompt:
You are an expert agricultural advisor specializing in crop diseases for Indian smallholder farmers. You have deep knowledge of tomato, wheat, and rice diseases common in India. You provide practical, cost-effective advice in simple Hindi.

User Prompt Template:
<image>{base64_encoded_image}</image>

Farmer Context:
- Location: {location}
- Previous diseases: {disease_history}
- Current crops: {crops}

Farmer's description: "{voice_description}"

Task:
1. Identify the crop type (tomato/wheat/rice)
2. Diagnose any visible diseases with confidence scores
3. If image quality is poor, explain what's needed
4. Provide 2-3 treatment options (prioritize organic)
5. Include cost estimates in rupees per acre
6. Consider weather: {weather_forecast}

Respond in Hindi with this structure:
फसल: [crop name]
बीमारी: [disease name in Hindi and English]
विश्वास स्तर: [confidence %]
उपचार विकल्प:
1. [organic option with cost]
2. [chemical option with cost]
सावधानियां: [precautions]
```

**Disease Library** (Minimum Coverage):
- Tomato: Early Blight, Late Blight, Leaf Curl, Septoria Leaf Spot, Bacterial Spot, Fusarium Wilt, Verticillium Wilt, Tomato Mosaic Virus, Powdery Mildew, Anthracnose, Blossom End Rot, Leaf Miner, Fruit Borer, Whitefly, Aphids
- Wheat: Rust (Yellow/Brown/Black), Powdery Mildew, Loose Smut, Karnal Bunt, Fusarium Head Blight, Septoria Leaf Blotch, Tan Spot, Aphids, Army Worm, Termites
- Rice: Blast, Bacterial Leaf Blight, Sheath Blight, Brown Spot, Tungro Virus, Stem Borer, Leaf Folder, Gall Midge, Plant Hopper, Bacterial Leaf Streak, False Smut, Bakanae


### Component 4: Market Intelligence Module

**Responsibility**: Fetch and analyze market prices, provide selling recommendations

**Interface**:
```python
class MarketIntelligenceModule:
    def get_market_prices(
        commodity: str,
        farmer_location: tuple[float, float],
        radius_km: int = 50
    ) -> list[MandiPrice]:
        """
        Fetch current prices from e-NAM API
        
        Returns list of MandiPrice:
          - mandi_name: str
          - district: str
          - price: float (₹/quintal)
          - arrival_quantity: float (quintals)
          - distance_km: float
          - last_updated: datetime
        """
    
    def calculate_price_trend(
        commodity: str,
        mandi: str,
        days: int = 3
    ) -> PriceTrend:
        """
        Calculate price trend from cached historical data
        
        Returns:
            PriceTrend:
              - current_price: float
              - moving_average: float
              - trend_direction: str (up/down/stable)
              - volatility: float
              - prediction_confidence: float
        """
    
    def generate_selling_recommendation(
        prices: list[MandiPrice],
        trends: list[PriceTrend],
        weather: WeatherForecast,
        crop_quality: Optional[str],
        festivals: list[Festival]
    ) -> SellingRecommendation:
        """
        Generate AI-powered selling recommendation
        
        Returns:
            SellingRecommendation:
              - action: str (sell_now/wait/sell_partial)
              - recommended_mandi: str
              - expected_price: float
              - reasoning: str (in Hindi)
              - alternative_options: list[str] (e.g., ONDC)
        """
```

**e-NAM API Integration**:
```
Base URL: https://api.data.gov.in/resource/9ef84268-d588-465a-a308-a864a43d0070

Request Parameters:
  - api-key: {government_api_key}
  - format: json
  - filters[commodity]: tomato
  - filters[state]: Maharashtra
  - filters[district]: Pune
  - limit: 100

Response Structure:
{
  "records": [
    {
      "state": "Maharashtra",
      "district": "Pune",
      "market": "Pune",
      "commodity": "Tomato",
      "variety": "Hybrid",
      "arrival_date": "2024-01-15",
      "min_price": "2000",
      "max_price": "2500",
      "modal_price": "2200"
    }
  ]
}
```

**Caching Strategy**:
- Cache e-NAM responses in DynamoDB for 2 hours
- Use cached data if API is slow (>5 seconds) or unavailable
- Pre-fetch prices for top 20 commodities every hour (Lambda scheduled event)
- Store 7 days of historical data for trend calculation

**Bedrock Prompt for Market Recommendations**:
```
System Prompt:
You are a market intelligence advisor for Indian farmers. You analyze mandi prices, weather patterns, festivals, and transport costs to recommend optimal selling strategies. You explain complex market dynamics in simple Hindi.

User Prompt Template:
Farmer wants to sell: {commodity}
Farmer location: {location}
Crop quality: {quality_assessment_from_disease_module}

Current Market Data:
{mandi_1}: ₹{price_1}/quintal, {distance_1} km away, trend: {trend_1}
{mandi_2}: ₹{price_2}/quintal, {distance_2} km away, trend: {trend_2}
{mandi_3}: ₹{price_3}/quintal, {distance_3} km away, trend: {trend_3}

3-Day Price History:
{date_1}: ₹{price}
{date_2}: ₹{price}
{date_3}: ₹{price}

Weather Forecast (7 days): {weather}
Upcoming Festivals: {festivals}
Transport Cost: ₹{cost}/quintal

ONDC Digital Price: ₹{ondc_price}/quintal (if available)

Task:
1. Analyze price trends and predict next 3-5 days
2. Consider weather impact on supply/demand
3. Factor in festival demand spikes
4. Compare transport costs vs price differences
5. Recommend: sell now / wait X days / sell partial quantity
6. If ONDC offers 15%+ premium, recommend it

Respond in Hindi with:
सिफारिश: [action]
कारण: [reasoning in 2-3 sentences]
अनुमानित कीमत: [expected price]
वैकल्पिक विकल्प: [alternatives]
```

### Component 5: Voice Processing Pipeline

**Responsibility**: Convert voice to text and text to voice

**Interface**:
```python
class VoiceProcessor:
    def transcribe_audio(
        audio_url: str,
        language_code: str = "hi-IN"
    ) -> TranscriptionResult:
        """
        Transcribe voice message using Amazon Transcribe
        
        Returns:
            TranscriptionResult:
              - text: str
              - confidence: float
              - language_detected: str
              - duration_seconds: float
        """
    
    def synthesize_speech(
        text: str,
        language: str = "hi",
        voice_id: str = "Aditi"
    ) -> bytes:
        """
        Convert text to speech using Amazon Polly
        
        Returns:
          MP3 audio bytes
        """
    
    def should_use_voice_output(farmer_profile: FarmerProfile) -> bool:
        """
        Determine if voice output should be used based on farmer preference
        """
```

**Transcribe Configuration**:
```python
transcribe_config = {
    "LanguageCode": "hi-IN",
    "MediaFormat": "ogg",  # Twilio sends OGG
    "Settings": {
        "VocabularyName": "agricultural-terms-hindi",
        "ShowSpeakerLabels": False,
        "MaxSpeakerLabels": 1,
        "ChannelIdentification": False
    }
}

# Custom Vocabulary (agricultural terms)
custom_vocabulary = [
    "मंडी", "क्विंटल", "खरीफ", "रबी", "जायद",
    "टमाटर", "गेहूं", "धान", "कीटनाशक", "उर्वरक",
    "ई-नाम", "एमएसपी", "बुवाई", "कटाई"
]
```

**Polly Configuration**:
```python
polly_config = {
    "VoiceId": "Aditi",  # Hindi female neural voice
    "Engine": "neural",
    "LanguageCode": "hi-IN",
    "OutputFormat": "mp3",
    "SampleRate": "22050",
    "TextType": "text",
    "SpeechMarkTypes": []
}

# For numbers and prices, use SSML for proper pronunciation
ssml_template = """
<speak>
    <prosody rate="90%">
        {text}
        <break time="500ms"/>
        कीमत <say-as interpret-as="currency">INR {price}</say-as> प्रति क्विंटल
    </prosody>
</speak>
"""
```

**Voice Output Optimization**:
- Cache common responses (greetings, error messages) to reduce Polly costs
- Split responses >60 seconds into multiple messages
- Use text fallback if Polly fails or farmer prefers text
- Compress MP3 to 64 kbps for faster WhatsApp delivery

### Component 6: Response Generator (Bedrock Orchestration)

**Responsibility**: Coordinate Bedrock calls with proper context and prompt engineering

**Interface**:
```python
class ResponseGenerator:
    def generate_response(
        user_message: str,
        farmer_context: FarmerProfile,
        conversation_history: list[Interaction],
        module: str,
        additional_context: dict
    ) -> str:
        """
        Generate AI response using Bedrock
        
        Args:
            user_message: Transcribed or text message
            farmer_context: Farmer profile
            conversation_history: Recent interactions
            module: disease_detection / market_intel / general
            additional_context: Module-specific data (prices, diagnosis, etc.)
            
        Returns:
            Generated response in Hindi
        """
```

**Bedrock API Call Structure**:
```python
bedrock_request = {
    "modelId": "anthropic.claude-sonnet-4-20250514",
    "contentType": "application/json",
    "accept": "application/json",
    "body": json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 2048,
        "temperature": 0.3,
        "system": SYSTEM_PROMPT,
        "messages": [
            {
                "role": "user",
                "content": [
                    {
                        "type": "image",
                        "source": {
                            "type": "url",
                            "url": image_url
                        }
                    } if image_url else None,
                    {
                        "type": "text",
                        "text": user_prompt
                    }
                ]
            }
        ]
    })
}
```

**System Prompt (Master)**:
```
You are "Kisan AI Sahayak", an expert agricultural advisor for Indian smallholder farmers. You specialize in:
1. Crop disease diagnosis (tomato, wheat, rice)
2. Market intelligence and selling strategies
3. Practical, cost-effective farming advice

Guidelines:
- Always respond in Hindi (Devanagari script)
- Use simple language suitable for farmers with basic literacy
- Provide specific, actionable advice with costs in rupees
- Reference Indian agricultural calendar (kharif, rabi, zayad seasons)
- Use Indian units: acre, quintal, kg (not hectares or metric tons)
- Be empathetic and encouraging
- If uncertain, say so and suggest consulting local agricultural officer
- Never recommend unproven or dangerous treatments

Farmer Context:
Name: {name}
Location: {location}
Crops: {crops}
Previous diseases: {disease_history}
Last interaction: {last_interaction}

Conversation History:
{conversation_history}

Current Date: {current_date}
Season: {season}
```


## Data Models

### FarmerProfile
```python
from dataclasses import dataclass
from datetime import datetime
from typing import Optional

@dataclass
class FarmerProfile:
    phone: str  # E.164 format: +919876543210
    name: Optional[str] = None
    location: Optional[tuple[float, float]] = None  # (latitude, longitude)
    preferred_language: str = "hi"  # ISO 639-1 code
    crops: list[str] = None  # ["tomato", "wheat", "rice"]
    created_at: datetime = None
    last_interaction: datetime = None
    total_queries: int = 0
    premium_status: bool = False
    voice_preference: bool = True  # True = voice output, False = text only
    
    def to_dynamodb_item(self) -> dict:
        """Convert to DynamoDB item format"""
        return {
            "phone": self.phone,
            "name": self.name or "",
            "location": {
                "lat": self.location[0] if self.location else 0,
                "lon": self.location[1] if self.location else 0
            },
            "preferred_language": self.preferred_language,
            "crops": self.crops or [],
            "created_at": int(self.created_at.timestamp()),
            "last_interaction": int(self.last_interaction.timestamp()),
            "total_queries": self.total_queries,
            "premium_status": self.premium_status,
            "voice_preference": self.voice_preference
        }
```

### DiagnosisResult
```python
@dataclass
class Disease:
    name_english: str
    name_hindi: str
    confidence: float  # 0.0 to 1.0
    severity: str  # "mild", "moderate", "severe"
    
@dataclass
class DiagnosisResult:
    crop_type: str  # "tomato", "wheat", "rice"
    diseases: list[Disease]
    overall_confidence: float
    image_quality_score: float  # 0.0 to 1.0
    requires_better_image: bool
    analysis_timestamp: datetime
    image_s3_key: str
    
    def is_healthy(self) -> bool:
        """Check if crop is healthy (no diseases detected)"""
        return len(self.diseases) == 0
    
    def primary_disease(self) -> Optional[Disease]:
        """Get disease with highest confidence"""
        return max(self.diseases, key=lambda d: d.confidence) if self.diseases else None
```

### TreatmentPlan
```python
@dataclass
class Treatment:
    name: str
    type: str  # "organic", "chemical", "cultural"
    ingredients: list[str]
    dosage: str  # e.g., "250ml per 15 liters water"
    application_method: str  # "spray", "soil_application", "seed_treatment"
    cost_per_acre: float  # in rupees
    effectiveness: float  # 0.0 to 1.0
    safety_precautions: list[str]
    
@dataclass
class TreatmentPlan:
    diagnosis: DiagnosisResult
    treatments: list[Treatment]  # Sorted by priority (organic first)
    application_timing: str  # "immediate", "morning", "evening", "wait_for_weather"
    weather_considerations: str
    expected_recovery_days: int
    preventive_measures: list[str]
    follow_up_date: datetime
```

### MandiPrice
```python
@dataclass
class MandiPrice:
    mandi_name: str
    district: str
    state: str
    commodity: str
    variety: str
    price_min: float  # ₹/quintal
    price_max: float
    price_modal: float  # Most common price
    arrival_quantity: float  # quintals
    date: datetime
    distance_km: float  # From farmer location
    transport_cost: float  # Estimated ₹/quintal
    
    def net_price(self) -> float:
        """Price after transport cost"""
        return self.price_modal - self.transport_cost
```

### PriceTrend
```python
@dataclass
class PriceTrend:
    commodity: str
    mandi: str
    current_price: float
    moving_average_3day: float
    moving_average_7day: float
    trend_direction: str  # "up", "down", "stable"
    volatility: float  # Standard deviation
    prediction_next_3days: float
    prediction_confidence: float  # 0.0 to 1.0
    historical_prices: list[tuple[datetime, float]]  # Last 7 days
```

### SellingRecommendation
```python
@dataclass
class SellingRecommendation:
    action: str  # "sell_now", "wait_3_days", "wait_7_days", "sell_partial"
    recommended_mandi: str
    expected_price: float
    reasoning_hindi: str  # Detailed explanation in Hindi
    confidence: float
    alternative_options: list[dict]  # [{"option": "ONDC", "price": 2800, "pros": [...]}]
    risk_factors: list[str]  # ["weather_uncertainty", "festival_demand"]
    optimal_selling_date: Optional[datetime]
```

### Interaction
```python
@dataclass
class Interaction:
    interaction_id: str  # UUID
    phone: str
    timestamp: datetime
    user_message: str
    user_message_type: str  # "text", "voice", "image"
    system_response: str
    module_used: str  # "disease_detection", "market_intel", "general", "onboarding"
    media_urls: list[str]
    response_time_ms: int
    bedrock_tokens_used: int
    cost_rupees: float
```

### WeatherForecast
```python
@dataclass
class WeatherForecast:
    location: tuple[float, float]
    forecast_days: list[dict]  # [{date, temp_max, temp_min, rain_mm, humidity, wind_speed}]
    
    def is_rain_expected(self, within_hours: int = 24) -> bool:
        """Check if rain is expected within specified hours"""
        pass
    
    def is_suitable_for_spraying(self) -> bool:
        """Check if weather is suitable for pesticide application"""
        # No rain for 24h, wind < 10 km/h, temp < 35°C
        pass
```

### MessageContext
```python
from enum import Enum

class MessageType(Enum):
    TEXT = "text"
    VOICE = "voice"
    IMAGE = "image"
    LOCATION = "location"

@dataclass
class MessageContext:
    farmer_phone: str
    message_type: MessageType
    content: str  # Transcribed text or original text
    media_url: Optional[str] = None
    location: Optional[tuple[float, float]] = None
    timestamp: datetime = None
    twilio_message_sid: str = None
```


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property Reflection

After analyzing all acceptance criteria, I identified several areas of redundancy:

1. **Performance properties** (1.1, 2.1, 4.1, 6.1, 6.2, 9.1) can be consolidated into a single end-to-end latency property
2. **Accuracy properties** (1.2, 1.3, 2.2) are specific to ML models and should be tested separately but can share a common pattern
3. **Output format properties** (2.3, 3.2, 3.3, 4.3, 13.6) can be combined into comprehensive output validation properties
4. **Error handling properties** (12.1-12.7) share common fallback patterns and can be consolidated
5. **Data persistence properties** (5.2, 5.3, 5.4, 16.6, 17.2) follow a common "store then retrieve" pattern
6. **Localization properties** (13.1-13.6) can be combined into comprehensive language/format properties

The following properties represent the unique, non-redundant validation requirements:

### Voice Processing Properties

**Property 1: Voice transcription performance**
*For any* voice message under 60 seconds, transcription should complete within 10 seconds and return text with confidence scores.
**Validates: Requirements 1.1**

**Property 2: Hindi transcription accuracy**
*For any* Hindi voice message with clear audio, transcription accuracy should meet or exceed 85% when compared to ground truth.
**Validates: Requirements 1.2**

**Property 3: Long audio segmentation**
*For any* voice message exceeding 60 seconds, the system should split it into segments, process each segment, and combine results into coherent text.
**Validates: Requirements 1.5**

**Property 4: Voice synthesis completeness**
*For any* text response generated by the system, voice synthesis should produce valid audio output in Hindi with all text content represented.
**Validates: Requirements 7.1**

**Property 5: Voice message splitting**
*For any* text response that would exceed 60 seconds of speech, the system should split it into multiple voice messages, each under 60 seconds.
**Validates: Requirements 7.4**

**Property 6: Indian number pronunciation**
*For any* response containing prices or numbers, voice synthesis should use Indian numbering format (lakh, crore) in pronunciation.
**Validates: Requirements 7.6**

### Disease Detection Properties

**Property 7: Image analysis performance**
*For any* crop image upload, disease detection should complete analysis and return results within 30 seconds.
**Validates: Requirements 2.1**

**Property 8: Disease detection confidence**
*For any* image containing a recognizable disease from the supported library, detection confidence should be 90% or higher.
**Validates: Requirements 2.2**

**Property 9: Bilingual disease naming**
*For any* identified disease, the response should include both Hindi and English names.
**Validates: Requirements 2.3**

**Property 10: Multiple disease detection**
*For any* image containing multiple diseases, all diseases should be listed with individual confidence scores.
**Validates: Requirements 2.4**

**Property 11: Poor image quality handling**
*For any* image with quality score below threshold, the system should reject analysis and provide specific guidance on lighting, distance, and focus.
**Validates: Requirements 2.5**

**Property 12: Unsupported crop handling**
*For any* image of crops other than Tomato, Wheat, or Rice, the system should inform the farmer and list supported crops.
**Validates: Requirements 2.6**

**Property 13: Healthy crop confirmation**
*For any* crop image with no detectable diseases, the system should confirm health status and provide preventive measures.
**Validates: Requirements 2.7**

**Property 14: Crop identification precedence**
*For any* uploaded image, crop type identification should complete before disease detection begins.
**Validates: Requirements 8.1**

### Treatment Advisory Properties

**Property 15: Minimum treatment options**
*For any* diagnosed disease, the system should provide at least 2 treatment options including at least one organic option.
**Validates: Requirements 3.1**

**Property 16: Treatment information completeness**
*For any* treatment recommendation, it should include cost per acre, application method, dosage, and safety precautions (if applicable).
**Validates: Requirements 3.2, 3.3, 3.7**

**Property 17: Weather-aware treatment timing**
*For any* treatment recommendation, if weather conditions affect application (rain, high temperature, wind), the system should specify optimal timing.
**Validates: Requirements 3.4, 3.5, 14.2, 14.3, 14.4**

**Property 18: Organic treatment prioritization**
*For any* disease with multiple treatment options, organic treatments should appear before chemical treatments in the recommendation list.
**Validates: Requirements 3.6**

### Market Intelligence Properties

**Property 19: Market price retrieval performance**
*For any* market price request, data should be fetched from e-NAM API or cache within 5 seconds.
**Validates: Requirements 4.1**

**Property 20: Nearest mandi selection**
*For any* farmer location and commodity, the system should return exactly the 3 nearest mandis sorted by distance.
**Validates: Requirements 4.2**

**Property 21: Price data completeness**
*For any* mandi price display, it should include current price, 3-day moving average, and trend direction (up/down/stable).
**Validates: Requirements 4.3**

**Property 22: Upward trend recommendation**
*For any* commodity with upward price trend, the system should recommend waiting to sell (unless crop condition prevents waiting).
**Validates: Requirements 4.4**

**Property 23: Downward trend recommendation**
*For any* commodity with downward price trend, the system should recommend immediate selling.
**Validates: Requirements 4.5**

**Property 24: Festival price prediction**
*For any* price query within 7 days of a major festival, the system should factor festival demand into price predictions.
**Validates: Requirements 4.6**

**Property 25: Transport cost optimization**
*For any* mandi comparison where transport costs exceed 10% of crop value, the system should recommend the local mandi over distant markets.
**Validates: Requirements 4.7**

**Property 26: ONDC price comparison**
*For any* commodity available on ONDC, the system should compare ONDC prices with mandi prices and suggest ONDC if it offers better value.
**Validates: Requirements 4.8, 18.1**

**Property 27: ONDC premium threshold**
*For any* commodity where ONDC offers 15% or higher price premium, the system should explicitly recommend digital selling with process explanation.
**Validates: Requirements 18.2, 18.3**

### Conversation Memory Properties

**Property 28: Profile creation on first interaction**
*For any* farmer phone number not in the database, the first interaction should create a new FarmerProfile record.
**Validates: Requirements 5.1**

**Property 29: Data persistence round-trip**
*For any* farmer data (location, disease diagnosis, treatment recommendation), storing it should allow retrieval of equivalent data in subsequent queries.
**Validates: Requirements 5.2, 5.3, 5.4**

**Property 30: Conversation context retrieval**
*For any* follow-up question from a farmer, the system should retrieve and include context from the last 5 interactions.
**Validates: Requirements 5.5**

**Property 31: History-aware recommendations**
*For any* advice generation, the AI should reference the farmer's crop history and past treatments in the response.
**Validates: Requirements 5.6**

**Property 32: Recurring disease detection**
*For any* disease that appears in a farmer's history within the last 90 days, the system should alert about recurrence and suggest preventive measures.
**Validates: Requirements 5.7**

### Interface and Integration Properties

**Property 33: End-to-end response time**
*For any* farmer request (text, voice, or image), the complete system response should be delivered within 30 seconds.
**Validates: Requirements 6.2, 9.1**

**Property 34: Multi-format input support**
*For any* message type (text, voice, image), the system should successfully parse and process it.
**Validates: Requirements 6.3**

**Property 35: Message ordering guarantee**
*For any* sequence of messages from a farmer, the system should process and respond to them in the order they were received.
**Validates: Requirements 6.5**

### Error Handling and Fallback Properties

**Property 36: Transcription failure recovery**
*For any* voice message where transcription fails or confidence is below 70%, the system should request resend or text input.
**Validates: Requirements 1.4, 12.4**

**Property 37: AI engine fallback**
*For any* request where Bedrock fails to generate a response, the system should provide a fallback message with support contact information.
**Validates: Requirements 12.1**

**Property 38: API unavailability fallback**
*For any* external API (e-NAM, weather) that is unavailable, the system should use cached data and inform the farmer of data age.
**Validates: Requirements 9.3, 12.2, 14.6**

**Property 39: Exponential backoff retry**
*For any* AWS service experiencing latency or errors, the system should implement exponential backoff with maximum 3 retry attempts.
**Validates: Requirements 9.4**

**Property 40: Database unavailability degradation**
*For any* request when DynamoDB is unavailable, the system should process the request without memory context and log for later retry.
**Validates: Requirements 12.5**

**Property 41: Rate limit handling**
*For any* external API rate limit exceeded, the system should queue the request and inform the farmer of expected delay.
**Validates: Requirements 12.6**

**Property 42: Critical failure SMS fallback**
*For any* critical service failure preventing WhatsApp response, the system should send SMS with helpline number.
**Validates: Requirements 12.7, 19.1**

### Localization and Cultural Properties

**Property 43: Hindi primary language**
*For any* system-generated response, the primary language should be Hindi (Devanagari script) unless farmer preference specifies otherwise.
**Validates: Requirements 13.1**

**Property 44: Indian units and formatting**
*For any* response containing measurements or prices, the system should use Indian units (acre, quintal, kg) and Indian numbering format (lakh, crore, ₹).
**Validates: Requirements 13.2, 13.3**

**Property 45: Agricultural calendar references**
*For any* timing recommendations, the system should reference Indian agricultural calendar (kharif, rabi, zayad) and festivals.
**Validates: Requirements 13.4**

**Property 46: Code-mixing understanding**
*For any* farmer message containing mixed Hindi-English words, the system should correctly understand and respond appropriately.
**Validates: Requirements 13.5**

### Cost Optimization Properties

**Property 47: Per-farmer cost constraint**
*For any* farmer over a monthly period, average AWS service costs should not exceed ₹5 per farmer.
**Validates: Requirements 10.1**

**Property 48: Conversation history archival**
*For any* conversation data older than 90 days, the system should automatically archive it to reduce storage costs.
**Validates: Requirements 10.2**

**Property 49: Response caching**
*For any* common response pattern (greetings, error messages), the system should use cached voice output instead of regenerating.
**Validates: Requirements 10.3**

**Property 50: Image compression**
*For any* uploaded image, the system should compress it to maximum 1MB before sending to Bedrock for analysis.
**Validates: Requirements 10.4**

**Property 51: Inactive farmer cold storage**
*For any* farmer with no interactions for 30 days, their data should be moved to cold storage tier.
**Validates: Requirements 10.5**

### Security and Privacy Properties

**Property 52: Data deletion compliance**
*For any* farmer requesting data deletion, all personal data should be removed from all tables within 30 days.
**Validates: Requirements 11.3**

**Property 53: PII logging protection**
*For any* logged interaction, personally identifiable information should not appear in plain text.
**Validates: Requirements 11.5**

**Property 54: Unauthorized access lockdown**
*For any* detected unauthorized access attempt, the affected farmer account should be locked and administrators alerted.
**Validates: Requirements 11.7**

### Monitoring and Analytics Properties

**Property 55: Request logging completeness**
*For any* processed request, the system should log response time, services used, and success/failure status.
**Validates: Requirements 16.1**

**Property 56: Error logging with context**
*For any* error occurrence, the system should log error type, affected service, and full context for debugging.
**Validates: Requirements 16.2**

**Property 57: High usage alerting**
*For any* day where total requests exceed 1000, the system should send an alert to administrators.
**Validates: Requirements 16.3**

**Property 58: Performance degradation detection**
*For any* period where average response time exceeds 40 seconds, the system should trigger performance investigation.
**Validates: Requirements 16.4**

**Property 59: Accuracy monitoring**
*For any* period where disease detection accuracy drops below 85%, the system should flag for model retraining.
**Validates: Requirements 16.5**

### Treatment Outcome Tracking Properties

**Property 60: Treatment follow-up scheduling**
*For any* treatment recommendation, the system should schedule and send a follow-up message after 7 days asking about effectiveness.
**Validates: Requirements 17.1**

**Property 61: Outcome data persistence**
*For any* farmer-reported treatment outcome (success or failure), the system should store it with disease and treatment details.
**Validates: Requirements 17.2, 17.3**

**Property 62: Treatment effectiveness adaptation**
*For any* treatment with consistently poor outcomes (>70% failure rate over 10+ cases), the system should deprioritize it in future recommendations.
**Validates: Requirements 17.5**

### Onboarding and Guidance Properties

**Property 63: First-time user welcome**
*For any* farmer's first interaction, the system should send a welcome message in Hindi explaining capabilities with example queries.
**Validates: Requirements 15.1, 15.2**

**Property 64: Unclear message guidance**
*For any* message that cannot be clearly understood, the system should provide guidance on how to phrase questions.
**Validates: Requirements 15.3**

**Property 65: Image quality feedback**
*For any* poor-quality image upload, the system should provide specific tips on lighting, distance, and focus for better photos.
**Validates: Requirements 15.4**

**Property 66: First success encouragement**
*For any* farmer completing their first successful interaction, the system should encourage saving the WhatsApp number.
**Validates: Requirements 15.6**

### Data Portability Properties

**Property 67: Data export generation**
*For any* farmer requesting data export, the system should generate a PDF report containing complete crop history, disease timeline, treatments, and market transactions.
**Validates: Requirements 20.1, 20.2, 20.3**

**Property 68: Export delivery**
*For any* generated export report, the system should send it via WhatsApp as a downloadable PDF file.
**Validates: Requirements 20.4**

**Property 69: Export before deletion**
*For any* data deletion request, the system should generate and provide the export before proceeding with deletion.
**Validates: Requirements 20.5**

### Offline and SMS Fallback Properties

**Property 70: SMS price information**
*For any* SMS query to the helpline number, the system should respond with basic price information via SMS.
**Validates: Requirements 19.2**

**Property 71: Reconnection sync**
*For any* farmer reconnecting after offline period, the system should sync and deliver any pending responses in order.
**Validates: Requirements 19.3, 19.4**


## Error Handling

### Error Classification

**Transient Errors** (Retry with exponential backoff):
- AWS service throttling (Transcribe, Bedrock, Polly)
- Network timeouts to external APIs (e-NAM, Weather)
- DynamoDB provisioned throughput exceeded
- S3 slow down errors

**Permanent Errors** (Fallback immediately):
- Invalid image format or corrupted file
- Unsupported crop type
- Malformed API responses
- Authentication failures

**Degraded Operation Errors** (Continue with reduced functionality):
- DynamoDB unavailable → Process without memory context
- e-NAM API down → Use cached prices with disclaimer
- Weather API down → Provide treatment advice with weather disclaimer
- Polly unavailable → Send text-only response

### Error Handling Strategies

**Strategy 1: Exponential Backoff with Jitter**
```python
def retry_with_backoff(func, max_retries=3):
    for attempt in range(max_retries):
        try:
            return func()
        except ThrottlingException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = (2 ** attempt) + random.uniform(0, 1)
            time.sleep(wait_time)
```

**Strategy 2: Circuit Breaker Pattern**
```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitBreakerOpenError()
        
        try:
            result = func()
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
            raise
```

**Strategy 3: Graceful Degradation**
```python
def get_market_prices(commodity, location):
    try:
        # Try live API first
        prices = enam_api.get_prices(commodity, location, timeout=5)
        return prices, "live"
    except (TimeoutError, APIUnavailableError):
        # Fall back to cache
        cached_prices = dynamodb.get_cached_prices(commodity, location)
        if cached_prices and cached_prices.age_hours < 2:
            return cached_prices.data, "cached"
        else:
            # Last resort: return error with helpful message
            raise PriceDataUnavailableError(
                "मंडी की कीमतें अभी उपलब्ध नहीं हैं। कृपया कुछ समय बाद पुनः प्रयास करें।"
            )
```

**Strategy 4: Fallback Responses**
```python
FALLBACK_RESPONSES = {
    "bedrock_failure": "क्षमा करें, मैं अभी आपकी मदद नहीं कर पा रहा हूं। कृपया हमारी हेल्पलाइन पर संपर्क करें: 1800-XXX-XXXX",
    "image_analysis_failure": "फोटो का विश्लेषण नहीं हो पाया। कृपया एक स्पष्ट फोटो भेजें जिसमें:\n1. अच्छी रोशनी हो\n2. पत्तियां साफ दिखें\n3. कैमरा स्थिर हो",
    "transcription_failure": "आपकी आवाज़ स्पष्ट नहीं सुनाई दी। कृपया:\n1. शांत जगह से बोलें\n2. फोन के पास बोलें\n3. या टेक्स्ट मैसेज भेजें",
    "database_unavailable": "आपका अनुरोध प्राप्त हुआ है। हम जल्द ही जवाब देंगे।"
}
```

### Error Response Format

All error responses to farmers should follow this structure:
```python
@dataclass
class ErrorResponse:
    error_type: str  # For logging
    user_message_hindi: str  # What farmer sees
    suggested_action: str  # What farmer should do
    support_contact: Optional[str]  # When to show helpline
    
def format_error_for_farmer(error: Exception) -> ErrorResponse:
    if isinstance(error, ImageQualityError):
        return ErrorResponse(
            error_type="poor_image_quality",
            user_message_hindi="फोटो की गुणवत्ता कम है",
            suggested_action="कृपया अच्छी रोशनी में स्पष्ट फोटो भेजें",
            support_contact=None
        )
    elif isinstance(error, UnsupportedCropError):
        return ErrorResponse(
            error_type="unsupported_crop",
            user_message_hindi=f"क्षमा करें, हम अभी केवल टमाटर, गेहूं और धान के लिए सहायता करते हैं",
            suggested_action="कृपया इन फसलों में से किसी एक की फोटो भेजें",
            support_contact=None
        )
    # ... more error types
```

### Monitoring and Alerting

**CloudWatch Alarms**:
1. Lambda error rate > 5% → Alert immediately
2. Average response time > 40 seconds → Alert after 5 minutes
3. DynamoDB throttling > 10 requests/minute → Alert immediately
4. Bedrock API errors > 10% → Alert immediately
5. Daily cost > ₹1000 → Alert immediately

**Custom Metrics**:
- Disease detection accuracy (tracked via farmer feedback)
- Cache hit rate for prices and voice responses
- Average cost per farmer per month
- Farmer satisfaction score (from follow-up surveys)


## Testing Strategy

### Dual Testing Approach

This system requires both **unit tests** for specific scenarios and **property-based tests** for universal correctness guarantees. Unit tests validate concrete examples and edge cases, while property tests verify that properties hold across all possible inputs through randomization.

**Balance**: Focus property tests on universal properties (e.g., "all responses must be in Hindi"), and unit tests on specific examples (e.g., "tomato early blight diagnosis returns correct treatment") and integration points.

### Property-Based Testing Configuration

**Framework**: Use **Hypothesis** for Python (Lambda functions)

**Configuration**:
```python
from hypothesis import given, settings, strategies as st

# Global settings for all property tests
settings.register_profile("kisan_ai", 
    max_examples=100,  # Minimum 100 iterations per property
    deadline=30000,    # 30 second timeout per test
    print_blob=True    # Print failing examples
)
settings.load_profile("kisan_ai")
```

**Test Tagging**: Each property test must reference its design document property:
```python
@given(st.text(min_size=1, max_size=500))
@settings(max_examples=100)
def test_hindi_primary_language(user_message):
    """
    Feature: kisan-ai-sahayak, Property 43: Hindi primary language
    For any system-generated response, the primary language should be Hindi
    """
    response = generate_response(user_message, default_farmer_profile())
    assert is_devanagari_script(response), f"Response not in Hindi: {response}"
    assert contains_hindi_words(response), "Response lacks Hindi vocabulary"
```

### Property Test Examples

**Property 1: Voice transcription performance**
```python
@given(st.binary(min_size=1000, max_size=500000))  # Audio data
@settings(max_examples=100)
def test_voice_transcription_performance(audio_data):
    """
    Feature: kisan-ai-sahayak, Property 1: Voice transcription performance
    For any voice message under 60 seconds, transcription completes within 10 seconds
    """
    start_time = time.time()
    result = transcribe_audio(audio_data, language="hi-IN")
    elapsed = time.time() - start_time
    
    if result.duration_seconds <= 60:
        assert elapsed <= 10, f"Transcription took {elapsed}s for {result.duration_seconds}s audio"
    assert result.text is not None
    assert 0 <= result.confidence <= 1
```

**Property 15: Minimum treatment options**
```python
@given(st.sampled_from(["tomato_early_blight", "wheat_rust", "rice_blast"]))
@settings(max_examples=100)
def test_minimum_treatment_options(disease_id):
    """
    Feature: kisan-ai-sahayak, Property 15: Minimum treatment options
    For any diagnosed disease, system provides at least 2 treatments including organic
    """
    diagnosis = DiagnosisResult(
        crop_type=disease_id.split("_")[0],
        diseases=[Disease(name_english=disease_id, confidence=0.95)],
        overall_confidence=0.95
    )
    
    treatment_plan = generate_treatment_plan(diagnosis, mock_weather(), [])
    
    assert len(treatment_plan.treatments) >= 2, "Must have at least 2 treatments"
    organic_count = sum(1 for t in treatment_plan.treatments if t.type == "organic")
    assert organic_count >= 1, "Must have at least 1 organic treatment"
```

**Property 20: Nearest mandi selection**
```python
@given(
    st.tuples(st.floats(8.0, 35.0), st.floats(68.0, 97.0)),  # India lat/lon
    st.sampled_from(["tomato", "wheat", "rice"])
)
@settings(max_examples=100)
def test_nearest_mandi_selection(farmer_location, commodity):
    """
    Feature: kisan-ai-sahayak, Property 20: Nearest mandi selection
    For any farmer location and commodity, returns exactly 3 nearest mandis sorted by distance
    """
    mandis = get_market_prices(commodity, farmer_location, radius_km=100)
    
    assert len(mandis) == 3, f"Expected 3 mandis, got {len(mandis)}"
    
    # Verify sorted by distance
    distances = [m.distance_km for m in mandis]
    assert distances == sorted(distances), "Mandis not sorted by distance"
    
    # Verify all within radius
    assert all(d <= 100 for d in distances), "Mandi outside radius"
```

**Property 29: Data persistence round-trip**
```python
@given(
    st.text(min_size=10, max_size=13, alphabet=st.characters(whitelist_categories=("Nd",))),  # Phone
    st.tuples(st.floats(8.0, 35.0), st.floats(68.0, 97.0)),  # Location
    st.lists(st.sampled_from(["tomato", "wheat", "rice"]), min_size=1, max_size=3)  # Crops
)
@settings(max_examples=100)
def test_data_persistence_round_trip(phone, location, crops):
    """
    Feature: kisan-ai-sahayak, Property 29: Data persistence round-trip
    For any farmer data stored, retrieval returns equivalent data
    """
    # Store
    profile = FarmerProfile(
        phone=f"+91{phone}",
        location=location,
        crops=crops,
        created_at=datetime.now()
    )
    context_manager.update_profile(profile.phone, profile.to_dynamodb_item())
    
    # Retrieve
    retrieved = context_manager.get_farmer_profile(profile.phone)
    
    # Verify equivalence
    assert retrieved.phone == profile.phone
    assert retrieved.location == profile.location
    assert set(retrieved.crops) == set(profile.crops)
```

**Property 44: Indian units and formatting**
```python
@given(
    st.floats(min_value=1000, max_value=100000),  # Price in rupees
    st.floats(min_value=0.5, max_value=50.0)      # Area in acres
)
@settings(max_examples=100)
def test_indian_units_and_formatting(price, area):
    """
    Feature: kisan-ai-sahayak, Property 44: Indian units and formatting
    For any response with measurements/prices, uses Indian units and format
    """
    response = format_price_recommendation(price, area)
    
    # Check for Indian units
    assert any(unit in response for unit in ["acre", "एकड़", "quintal", "क्विंटल", "kg", "किलो"])
    
    # Check for rupee symbol
    assert "₹" in response or "रुपये" in response
    
    # Check for Indian numbering (lakh/crore for large numbers)
    if price >= 100000:
        assert any(word in response for word in ["lakh", "लाख", "crore", "करोड़"])
```

### Unit Test Examples

**Unit Test 1: Tomato Early Blight Detection**
```python
def test_tomato_early_blight_detection():
    """Test specific disease detection with known image"""
    image_path = "test_data/tomato_early_blight_sample.jpg"
    
    result = analyze_crop_image(image_path, default_farmer_profile(), None)
    
    assert result.crop_type == "tomato"
    assert len(result.diseases) > 0
    assert result.diseases[0].name_english == "Early Blight"
    assert result.diseases[0].confidence >= 0.90
    assert result.diseases[0].name_hindi == "अर्ली ब्लाइट"
```

**Unit Test 2: Price Trend Calculation**
```python
def test_price_trend_upward():
    """Test upward price trend detection"""
    historical_prices = [
        (datetime(2024, 1, 1), 2000),
        (datetime(2024, 1, 2), 2100),
        (datetime(2024, 1, 3), 2200)
    ]
    
    trend = calculate_price_trend("tomato", "pune", historical_prices)
    
    assert trend.trend_direction == "up"
    assert trend.moving_average_3day == 2100
    assert trend.prediction_next_3days > 2200
```

**Unit Test 3: Welcome Message Content**
```python
def test_first_time_user_welcome():
    """Test welcome message for new farmer"""
    phone = "+919876543210"
    
    # Ensure no existing profile
    dynamodb.delete_item(Key={"phone": phone})
    
    response = handle_message(MessageContext(
        farmer_phone=phone,
        message_type=MessageType.TEXT,
        content="नमस्ते",
        timestamp=datetime.now()
    ))
    
    assert "स्वागत" in response or "welcome" in response.lower()
    assert "टमाटर" in response  # Mentions supported crops
    assert "गेहूं" in response
    assert "धान" in response
    # Check for example queries
    assert any(word in response for word in ["उदाहरण", "example", "जैसे"])
```

**Unit Test 4: Error Handling - Poor Image Quality**
```python
def test_poor_image_quality_handling():
    """Test system response to low-quality image"""
    # Create a very blurry/dark image
    poor_image = create_degraded_image(blur_factor=10, brightness=0.2)
    
    result = analyze_crop_image(poor_image, default_farmer_profile(), None)
    
    assert result.requires_better_image == True
    assert result.image_quality_score < 0.5
    
    # Check that response includes guidance
    response = format_diagnosis_response(result)
    assert any(word in response for word in ["रोशनी", "light", "स्पष्ट", "clear"])
```

**Unit Test 5: Integration - End-to-End Disease Detection Flow**
```python
def test_end_to_end_disease_detection():
    """Integration test for complete disease detection flow"""
    # Simulate WhatsApp webhook payload
    webhook_payload = {
        "From": "whatsapp:+919876543210",
        "Body": "मेरे टमाटर के पत्ते पीले हो रहे हैं",
        "MediaUrl0": "https://example.com/tomato_leaf.jpg",
        "MessageSid": "SM123456"
    }
    
    # Process through Lambda handler
    response = lambda_handler({"body": json.dumps(webhook_payload)}, {})
    
    assert response["statusCode"] == 200
    
    # Verify response sent to Twilio
    sent_message = get_last_twilio_message("+919876543210")
    assert sent_message is not None
    assert "टमाटर" in sent_message.body
    assert any(disease in sent_message.body for disease in ["ब्लाइट", "मोज़ेक", "विल्ट"])
    
    # Verify data stored in DynamoDB
    profile = context_manager.get_farmer_profile("+919876543210")
    assert len(profile.disease_history) > 0
```

### Test Data Management

**Synthetic Data Generation**:
- Use PlantVillage dataset for disease images (50,000+ labeled images)
- Generate synthetic farmer profiles with realistic Indian names, locations
- Create mock e-NAM API responses based on historical data patterns
- Generate Hindi voice samples using Polly for transcription testing

**Test Fixtures**:
```python
@pytest.fixture
def default_farmer_profile():
    return FarmerProfile(
        phone="+919876543210",
        name="राज कुमार",
        location=(18.5204, 73.8567),  # Pune
        preferred_language="hi",
        crops=["tomato", "wheat"],
        created_at=datetime.now(),
        last_interaction=datetime.now()
    )

@pytest.fixture
def mock_weather():
    return WeatherForecast(
        location=(18.5204, 73.8567),
        forecast_days=[
            {"date": "2024-01-15", "temp_max": 28, "temp_min": 18, "rain_mm": 0, "humidity": 65, "wind_speed": 8},
            {"date": "2024-01-16", "temp_max": 30, "temp_min": 19, "rain_mm": 0, "humidity": 60, "wind_speed": 10},
        ]
    )
```

### Test Coverage Goals

- **Unit Test Coverage**: 80% code coverage minimum
- **Property Test Coverage**: 100% of correctness properties (71 properties)
- **Integration Test Coverage**: All critical user flows (disease detection, market query, onboarding)
- **Performance Test Coverage**: All latency requirements (<30s end-to-end)

### Continuous Testing

**Pre-deployment**:
- Run all unit tests and property tests in CI/CD pipeline
- Fail deployment if any test fails or coverage drops below 80%
- Run integration tests against staging environment

**Post-deployment**:
- Canary deployment to 5% of traffic
- Monitor error rates and response times
- Automated rollback if error rate > 5% or response time > 40s

**Production Monitoring**:
- Synthetic transaction tests every 5 minutes (disease detection, market query)
- Alert if synthetic tests fail 3 times consecutively
- Weekly accuracy validation using farmer feedback data

