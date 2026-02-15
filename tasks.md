# Implementation Plan: Kisan AI Sahayak

## Overview

This implementation plan breaks down the Kisan AI Sahayak system into discrete, incremental coding tasks. The system is a serverless AWS-based AI assistant for Indian smallholder farmers, providing crop disease detection, market intelligence, and conversational memory through WhatsApp.

The implementation follows a bottom-up approach: core data models → individual modules → integration → testing. Each task builds on previous work, with checkpoints to validate functionality before proceeding.

## Tasks

- [ ] 1. Set up project structure and AWS infrastructure
  - Create Python project with virtual environment
  - Set up AWS CDK or CloudFormation templates for Lambda, API Gateway, DynamoDB, S3
  - Configure environment variables and secrets management
  - Set up logging and monitoring with CloudWatch
  - _Requirements: 9.1, 16.1, 16.2_

- [ ] 2. Implement core data models and DynamoDB schemas
  - [ ] 2.1 Create data model classes (FarmerProfile, DiagnosisResult, TreatmentPlan, MandiPrice, etc.)
    - Implement all dataclasses from design document with validation
    - Add serialization methods (to_dynamodb_item, from_dynamodb_item)
    - _Requirements: 5.1, 5.2, 5.3_
  
  - [ ]* 2.2 Write property test for data persistence round-trip
    - **Property 29: Data persistence round-trip**
    - **Validates: Requirements 5.2, 5.3, 5.4**
  
  - [ ] 2.3 Create DynamoDB table schemas and access patterns
    - Define tables: FarmerProfiles, ConversationHistory, DiseaseRecords, PriceCache
    - Implement GSIs for location-based and time-based queries
    - Add TTL configuration for automatic data expiration
    - _Requirements: 5.1, 5.2, 10.2_
  
  - [ ]* 2.4 Write unit tests for data model validation
    - Test edge cases for phone number formats, location coordinates
    - Test serialization/deserialization correctness
    - _Requirements: 5.1, 5.2_

- [ ] 3. Implement Farmer Context Manager
  - [ ] 3.1 Create FarmerContextManager class with DynamoDB operations
    - Implement get_farmer_profile, update_profile, store_interaction
    - Implement get_conversation_history with pagination
    - Add error handling for DynamoDB throttling and unavailability
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_
  
  - [ ]* 3.2 Write property test for profile creation on first interaction
    - **Property 28: Profile creation on first interaction**
    - **Validates: Requirements 5.1**
  
  - [ ]* 3.3 Write property test for conversation context retrieval
    - **Property 30: Conversation context retrieval**
    - **Validates: Requirements 5.5**
  
  - [ ]* 3.4 Write unit tests for context manager edge cases
    - Test behavior when DynamoDB is unavailable
    - Test concurrent updates to same farmer profile
    - _Requirements: 5.1, 12.5_

- [ ] 4. Checkpoint - Verify data layer functionality
  - Ensure all tests pass for data models and context manager
  - Verify DynamoDB tables are created correctly
  - Test manual CRUD operations against DynamoDB
  - Ask the user if questions arise

- [ ] 5. Implement Voice Processing Pipeline
  - [ ] 5.1 Create VoiceProcessor class with Transcribe integration
    - Implement transcribe_audio with custom vocabulary for agricultural terms
    - Add audio format conversion (OGG to formats Transcribe accepts)
    - Implement confidence score validation and retry logic
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_
  
  - [ ] 5.2 Implement Polly integration for voice synthesis
    - Implement synthesize_speech with Hindi voice (Aditi)
    - Add SSML formatting for numbers and prices
    - Implement response caching for common phrases
    - Add voice message splitting for responses >60 seconds
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.6_
  
  - [ ]* 5.3 Write property test for voice transcription performance
    - **Property 1: Voice transcription performance**
    - **Validates: Requirements 1.1**
  
  - [ ]* 5.4 Write property test for voice message splitting
    - **Property 5: Voice message splitting**
    - **Validates: Requirements 7.4**
  
  - [ ]* 5.5 Write property test for Indian number pronunciation
    - **Property 6: Indian number pronunciation**
    - **Validates: Requirements 7.6**
  
  - [ ]* 5.6 Write unit tests for voice processing edge cases
    - Test handling of background noise
    - Test transcription failure recovery
    - Test Polly unavailability fallback
    - _Requirements: 1.4, 1.6, 12.1_

- [ ] 6. Implement Disease Detection Module
  - [ ] 6.1 Create DiseaseDetectionModule class with Bedrock integration
    - Implement analyze_crop_image with image upload to S3
    - Create prompt templates for disease detection
    - Implement image quality assessment
    - Add crop type identification logic
    - Parse Bedrock responses into DiagnosisResult objects
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 2.7, 8.1_
  
  - [ ] 6.2 Implement treatment plan generation
    - Create generate_treatment_plan with weather integration
    - Build treatment library for tomato, wheat, rice diseases
    - Implement organic treatment prioritization
    - Add cost estimation logic
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7_
  
  - [ ]* 6.3 Write property test for image analysis performance
    - **Property 7: Image analysis performance**
    - **Validates: Requirements 2.1**
  
  - [ ]* 6.4 Write property test for bilingual disease naming
    - **Property 9: Bilingual disease naming**
    - **Validates: Requirements 2.3**
  
  - [ ]* 6.5 Write property test for minimum treatment options
    - **Property 15: Minimum treatment options**
    - **Validates: Requirements 3.1**
  
  - [ ]* 6.6 Write property test for organic treatment prioritization
    - **Property 18: Organic treatment prioritization**
    - **Validates: Requirements 3.6**
  
  - [ ]* 6.7 Write property test for weather-aware treatment timing
    - **Property 17: Weather-aware treatment timing**
    - **Validates: Requirements 3.4, 3.5, 14.2, 14.3, 14.4**
  
  - [ ]* 6.8 Write unit tests for disease detection scenarios
    - Test tomato early blight detection with sample image
    - Test wheat rust detection with sample image
    - Test rice blast detection with sample image
    - Test poor image quality rejection
    - Test unsupported crop handling
    - _Requirements: 2.1, 2.2, 2.5, 2.6_

- [ ] 7. Implement Market Intelligence Module
  - [ ] 7.1 Create MarketIntelligenceModule class with e-NAM API integration
    - Implement get_market_prices with location-based filtering
    - Add distance calculation from farmer location
    - Implement price caching in DynamoDB
    - Add error handling for API unavailability
    - _Requirements: 4.1, 4.2, 9.3, 12.2_
  
  - [ ] 7.2 Implement price trend calculation
    - Create calculate_price_trend with moving averages
    - Implement trend direction detection (up/down/stable)
    - Add volatility calculation
    - _Requirements: 4.3_
  
  - [ ] 7.3 Implement selling recommendation generation
    - Create generate_selling_recommendation with Bedrock
    - Integrate weather forecast for timing recommendations
    - Add festival calendar for demand prediction
    - Implement transport cost optimization
    - Add ONDC price comparison
    - _Requirements: 4.4, 4.5, 4.6, 4.7, 4.8, 18.1, 18.2, 18.3_
  
  - [ ]* 7.4 Write property test for market price retrieval performance
    - **Property 19: Market price retrieval performance**
    - **Validates: Requirements 4.1**
  
  - [ ]* 7.5 Write property test for nearest mandi selection
    - **Property 20: Nearest mandi selection**
    - **Validates: Requirements 4.2**
  
  - [ ]* 7.6 Write property test for price data completeness
    - **Property 21: Price data completeness**
    - **Validates: Requirements 4.3**
  
  - [ ]* 7.7 Write property test for upward trend recommendation
    - **Property 22: Upward trend recommendation**
    - **Validates: Requirements 4.4**
  
  - [ ]* 7.8 Write property test for transport cost optimization
    - **Property 25: Transport cost optimization**
    - **Validates: Requirements 4.7**
  
  - [ ]* 7.9 Write unit tests for market intelligence scenarios
    - Test price trend calculation with upward prices
    - Test price trend calculation with downward prices
    - Test festival impact on recommendations
    - Test e-NAM API failure fallback to cache
    - _Requirements: 4.3, 4.4, 4.5, 4.6, 12.2_

- [ ] 8. Checkpoint - Verify module functionality
  - Ensure all tests pass for voice, disease detection, and market modules
  - Test each module independently with mock data
  - Verify Bedrock prompts produce expected outputs
  - Ask the user if questions arise

- [ ] 9. Implement Response Generator and Bedrock orchestration
  - [ ] 9.1 Create ResponseGenerator class
    - Implement generate_response with context assembly
    - Create system prompt templates for different modules
    - Implement conversation history formatting
    - Add response parsing and validation
    - _Requirements: 5.6, 13.1, 13.2, 13.3, 13.4, 13.5, 13.6_
  
  - [ ]* 9.2 Write property test for Hindi primary language
    - **Property 43: Hindi primary language**
    - **Validates: Requirements 13.1**
  
  - [ ]* 9.3 Write property test for Indian units and formatting
    - **Property 44: Indian units and formatting**
    - **Validates: Requirements 13.2, 13.3**
  
  - [ ]* 9.4 Write property test for code-mixing understanding
    - **Property 46: Code-mixing understanding**
    - **Validates: Requirements 13.5**
  
  - [ ]* 9.5 Write unit tests for response generation
    - Test response with farmer history context
    - Test response with code-mixed input
    - Test response formatting with prices and measurements
    - _Requirements: 5.6, 13.1, 13.5_

- [ ] 10. Implement Message Router and WhatsApp integration
  - [ ] 10.1 Create MessageRouter class
    - Implement route_message to parse Twilio webhook payloads
    - Add Twilio signature validation for security
    - Implement message type detection (text/voice/image/location)
    - Add media URL handling and S3 upload
    - _Requirements: 6.1, 6.3, 6.4_
  
  - [ ]* 10.2 Write property test for multi-format input support
    - **Property 34: Multi-format input support**
    - **Validates: Requirements 6.3**
  
  - [ ]* 10.3 Write unit tests for message routing
    - Test text message parsing
    - Test voice message parsing
    - Test image message parsing
    - Test Twilio signature validation
    - _Requirements: 6.1, 6.3, 11.7_

- [ ] 11. Implement Lambda Orchestrator function
  - [ ] 11.1 Create main Lambda handler
    - Implement lambda_handler entry point
    - Add request routing to appropriate modules
    - Implement error handling and fallback logic
    - Add response formatting for Twilio
    - Integrate all modules (context, voice, disease, market, response)
    - _Requirements: 6.2, 9.1, 12.1, 12.2, 12.3, 12.4, 12.5, 12.6, 12.7_
  
  - [ ] 11.2 Implement error handling strategies
    - Add exponential backoff with jitter for retries
    - Implement circuit breaker pattern for external APIs
    - Add graceful degradation for service unavailability
    - Create fallback response templates
    - _Requirements: 9.4, 12.1, 12.2, 12.3, 12.4, 12.5, 12.6, 12.7_
  
  - [ ]* 11.3 Write property test for end-to-end response time
    - **Property 33: End-to-end response time**
    - **Validates: Requirements 6.2, 9.1**
  
  - [ ]* 11.4 Write property test for message ordering guarantee
    - **Property 35: Message ordering guarantee**
    - **Validates: Requirements 6.5**
  
  - [ ]* 11.5 Write property test for API unavailability fallback
    - **Property 38: API unavailability fallback**
    - **Validates: Requirements 9.3, 12.2, 14.6**
  
  - [ ]* 11.6 Write integration tests for end-to-end flows
    - Test complete disease detection flow (voice + image → diagnosis → treatment)
    - Test complete market intelligence flow (text query → prices → recommendation)
    - Test onboarding flow for new farmer
    - Test error scenarios (Bedrock failure, DynamoDB unavailable, etc.)
    - _Requirements: 6.2, 9.1, 12.1, 12.5, 15.1_

- [ ] 12. Implement onboarding and user guidance
  - [ ] 12.1 Create onboarding logic for new farmers
    - Implement first-time user detection
    - Create welcome message template in Hindi
    - Add example queries and capability explanation
    - _Requirements: 15.1, 15.2, 15.5_
  
  - [ ] 12.2 Implement guidance for unclear messages
    - Add message clarity detection
    - Create guidance templates for common issues
    - Implement image quality feedback
    - _Requirements: 15.3, 15.4_
  
  - [ ]* 12.3 Write property test for first-time user welcome
    - **Property 63: First-time user welcome**
    - **Validates: Requirements 15.1, 15.2**
  
  - [ ]* 12.4 Write property test for image quality feedback
    - **Property 65: Image quality feedback**
    - **Validates: Requirements 15.4**
  
  - [ ]* 12.5 Write unit tests for onboarding scenarios
    - Test welcome message content and format
    - Test unclear message guidance
    - Test poor image quality feedback
    - _Requirements: 15.1, 15.3, 15.4_

- [ ] 13. Implement treatment outcome tracking
  - [ ] 13.1 Create follow-up scheduling system
    - Implement treatment follow-up scheduling (7 days after recommendation)
    - Create follow-up message templates
    - Add outcome recording logic
    - _Requirements: 17.1, 17.2, 17.3_
  
  - [ ] 13.2 Implement treatment effectiveness adaptation
    - Add outcome analysis logic
    - Implement treatment deprioritization for poor outcomes
    - _Requirements: 17.4, 17.5_
  
  - [ ]* 13.3 Write property test for treatment follow-up scheduling
    - **Property 60: Treatment follow-up scheduling**
    - **Validates: Requirements 17.1**
  
  - [ ]* 13.4 Write property test for outcome data persistence
    - **Property 61: Outcome data persistence**
    - **Validates: Requirements 17.2, 17.3**
  
  - [ ]* 13.5 Write unit tests for outcome tracking
    - Test follow-up message delivery
    - Test outcome recording
    - Test treatment effectiveness calculation
    - _Requirements: 17.1, 17.2, 17.5_

- [ ] 14. Implement data export and portability
  - [ ] 14.1 Create data export functionality
    - Implement PDF report generation with farmer history
    - Add disease timeline, treatments, and market transactions
    - Create export delivery via WhatsApp
    - _Requirements: 20.1, 20.2, 20.3, 20.4_
  
  - [ ] 14.2 Implement data deletion with export
    - Add export-before-deletion logic
    - Implement complete data removal across all tables
    - _Requirements: 11.3, 20.5_
  
  - [ ]* 14.3 Write property test for data export generation
    - **Property 67: Data export generation**
    - **Validates: Requirements 20.1, 20.2, 20.3**
  
  - [ ]* 14.4 Write property test for data deletion compliance
    - **Property 52: Data deletion compliance**
    - **Validates: Requirements 11.3**
  
  - [ ]* 14.5 Write unit tests for data portability
    - Test PDF report generation
    - Test export delivery
    - Test data deletion after export
    - _Requirements: 20.1, 20.4, 11.3_

- [ ] 15. Checkpoint - Verify complete system integration
  - Ensure all tests pass (unit and property tests)
  - Test complete user journeys end-to-end
  - Verify all AWS services are properly configured
  - Test error handling and fallback scenarios
  - Ask the user if questions arise

- [ ] 16. Implement monitoring and cost optimization
  - [ ] 16.1 Set up CloudWatch alarms and custom metrics
    - Create alarms for error rate, response time, throttling
    - Implement custom metrics for accuracy, cost per farmer
    - Add alerting for high usage and performance degradation
    - _Requirements: 16.1, 16.2, 16.3, 16.4, 16.5_
  
  - [ ] 16.2 Implement cost optimization features
    - Add response caching for common phrases
    - Implement image compression before Bedrock
    - Add inactive farmer cold storage migration
    - Implement conversation history archival
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5_
  
  - [ ]* 16.3 Write property test for per-farmer cost constraint
    - **Property 47: Per-farmer cost constraint**
    - **Validates: Requirements 10.1**
  
  - [ ]* 16.4 Write property test for image compression
    - **Property 50: Image compression**
    - **Validates: Requirements 10.4**
  
  - [ ]* 16.5 Write unit tests for monitoring
    - Test CloudWatch metric publishing
    - Test alarm triggering conditions
    - Test cost tracking accuracy
    - _Requirements: 16.1, 16.2, 16.3_

- [ ] 17. Implement security and privacy features
  - [ ] 17.1 Add security controls
    - Implement IAM roles with least privilege
    - Add DynamoDB encryption at rest
    - Implement TLS for all data in transit
    - Add PII logging protection
    - _Requirements: 11.1, 11.2, 11.4, 11.5_
  
  - [ ] 17.2 Implement privacy features
    - Add consent management for image usage
    - Implement unauthorized access detection
    - _Requirements: 11.6, 11.7_
  
  - [ ]* 17.3 Write property test for PII logging protection
    - **Property 53: PII logging protection**
    - **Validates: Requirements 11.5**
  
  - [ ]* 17.4 Write unit tests for security
    - Test IAM role permissions
    - Test encryption configuration
    - Test unauthorized access detection
    - _Requirements: 11.1, 11.4, 11.7_

- [ ] 18. Implement offline capability and SMS fallback
  - [ ] 18.1 Create SMS fallback system
    - Implement SMS sending for WhatsApp unavailability
    - Add basic price information via SMS
    - Implement reconnection sync logic
    - _Requirements: 19.1, 19.2, 19.3, 19.4_
  
  - [ ]* 18.2 Write property test for SMS price information
    - **Property 70: SMS price information**
    - **Validates: Requirements 19.2**
  
  - [ ]* 18.3 Write property test for reconnection sync
    - **Property 71: Reconnection sync**
    - **Validates: Requirements 19.3, 19.4**
  
  - [ ]* 18.4 Write unit tests for offline capability
    - Test SMS fallback triggering
    - Test message queuing during offline
    - Test sync on reconnection
    - _Requirements: 19.1, 19.3, 19.4_

- [ ] 19. Deploy to AWS and configure production environment
  - Deploy Lambda functions with proper IAM roles
  - Configure API Gateway with Twilio webhook
  - Set up DynamoDB tables with proper indexes
  - Configure S3 bucket with lifecycle policies
  - Set up CloudWatch dashboards and alarms
  - Configure Twilio WhatsApp Business API webhook
  - Test production deployment with synthetic transactions
  - _Requirements: 9.1, 9.2, 16.1, 16.2_

- [ ] 20. Final checkpoint - Production readiness validation
  - Run complete test suite (unit + property + integration)
  - Verify all 71 correctness properties pass
  - Test with real farmer scenarios (pilot group)
  - Validate cost per farmer is under ₹5/month
  - Verify response times are under 30 seconds
  - Confirm 99% uptime during business hours
  - Ask the user if questions arise

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation before proceeding
- Property tests validate universal correctness properties (71 total)
- Unit tests validate specific examples, edge cases, and integration points
- The implementation uses Python 3.11 for all Lambda functions
- AWS services: Lambda, API Gateway, DynamoDB, S3, Transcribe, Bedrock, Polly
- External integrations: Twilio WhatsApp API, e-NAM API, OpenWeather API
