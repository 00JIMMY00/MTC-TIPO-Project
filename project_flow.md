# Detailed System Design: AI-Driven Elbow Splint

This document expands on the high-level data flow, providing a granular view of the logic within the microcontroller and the API interactions with the cloud.

## 1. High-Level System Interaction

(Overview of the complete loop)

```mermaid
sequenceDiagram
    autonumber
    participant P as Patient
    participant S as Sensors (EMG, IMU, Encoder)
    participant MCU as Microcontroller (Device)
    participant M as Stepper Motors
    participant C as Cloud (AI/Server)

    Note over P, M: Real-time Operation Loop
    P->>S: Physical Interaction (Muscle signals, Movement)
    S->>MCU: Send Raw Data (EMG, Acceleration, Position)
    MCU->>MCU: Process Data (Filtering, Feature Extraction)
    MCU->>M: Real-time Assistive Control
    M->>P: Mechanical Assistance (Torque/Position)

    Note over MCU, C: Daily Data Sync & Refinement
    MCU->>MCU: Buffer/Log Daily Session Data
  
    loop Daily Upload
        MCU->>C: Push Aggregated Session Data (Compressed)
    end

    Note over C: Cloud Processing
    C->>C: Analyze Patient Progress (ROM, Muscle Fatigue)
    C->>C: Refine Control Model / Calculate Parameters
    note right of C: Determine "Hardness" (Stiffness) <br/>or "Softness" (Compliance)

    Note over C, MCU: Feedback Loop
    C->>MCU: Send Updated Configuration (JSON)
    note right of MCU: { "stiffness": 0.5, "max_torque": 2.0, "mode": "adaptive" }

    Note over MCU, M: Adjustment
    MCU->>MCU: Update Control Algorithm Parameters
    MCU->>M: Apply New Control Strategy (Adjusted Hard/Soft Response)
    M->>P: Optimized Treatment
```

## 2. Detailed Logic Flowchart

This flowchart illustrates the internal decision-making process of the Microcontroller (MCU), separating the 100Hz real-time control loop from the daily cloud synchronization routine.

```mermaid
graph TD
    Start([Power On]) --> Init[Initialize Sensors & Motors]
    Init --> LoadConfig[Load Last Config from EEPROM]
    LoadConfig --> ReadSensors

    %% === Real-Time Loop ===
    subgraph RealTimeLoop ["Real-Time Control Loop (100Hz)"]
        ReadSensors[Read EMG, IMU, Encoder] --> Filter[Signal Processing & Filtering]
        Filter --> FeatureEx[Extract Features: RMS, Angle, Velocity]
        FeatureEx --> Safety{Safety Check?}
        Safety -- Unsafe --> EStop[Emergency Stop]
        Safety -- Safe --> ControlAlgo[Calculate Motor Torque]
        ControlAlgo --> MotorDrv[Drive Stepper Motor]
        MotorDrv --> LogData[Buffer Data to SD/Flash]
    end

    LogData --> EndSession{Session Ended?}
    EndSession -- No --> ReadSensors

    %% === Cloud Sync ===
    subgraph CloudSync ["Daily Cloud Synchronization"]
        ConnectNet[Connect to Wi-Fi/LTE]
        ConnectNet --> Auth[Authenticate with Cloud]
        Auth --> Compress[Compress Session Logs]
        Compress --> Upload[POST /api/v1/data]
        Upload --> WaitResp{Response 200 OK?}
        WaitResp -- No --> Retry[Retry Later]
        WaitResp -- Yes --> ParseJSON[Parse JSON Response]
        ParseJSON --> ExtractParams[Extract: Stiffness, Damping, ROM_Limits]
        ExtractParams --> Validate[Validate Parameters]
        Validate --> SaveConfig[Save New Config to EEPROM]
    end

    EndSession -- Yes --> ConnectNet
    SaveConfig --> Sleep([Deep Sleep / Charge Mode])
    Retry --> Sleep

```

## 3. Detailed Data Sequence & API Structure

This sequence diagram details the specific API calls, JSON payloads, and backend processing logic used to refine the patient's treatment plan.

```mermaid
sequenceDiagram
    participant MCU as Microcontroller
    participant API as Cloud API Gateway
    participant DB as Database
    participant AI as AI Model Engine

    Note over MCU: Session Complete. <br/>Preparing Data Payload.

    MCU->>MCU: Serialize Data
    note right of MCU: Payload: {<br/> "device_id": "SPLINT_01",<br/> "session_date": "2023-10-27",<br/> "metrics": {<br/>   "avg_rom": 120,<br/>   "spasticity_count": 3,<br/>   "muscle_fatigue_index": 0.8<br/> },<br/> "raw_samples_ref": "s3_blob_link"<br/>}

    MCU->>API: POST /telemetry/daily
    API->>DB: Store Session Metadata
    API-->>MCU: 202 Accepted

    Note over DB, AI: Async Processing Trigger
    DB->>AI: Trigger Retraining / Analysis
    AI->>AI: Fetch Historical Patient Data
    AI->>AI: Run Classification (Random Forest/LSTM)
  
    alt Patient Improving
        AI->>AI: Decrease Assistance (Lower K_stiff)
        note right of AI: Goal: Encourage active movement
    else Patient Fatigued/Regression
        AI->>AI: Increase Assistance (Higher K_stiff)
        note right of AI: Goal: Provide support
    end

    AI->>DB: Save New Control Parameters
  
    Note over MCU: Periodic Check or WebSocket
    MCU->>API: GET /config/latest?device_id=SPLINT_01
    API->>DB: Query Latest Config
    DB-->>API: Return Config
    API-->>MCU: 200 OK (JSON)
  
    note left of MCU: Response: {<br/> "param_version": 45,<br/> "control_mode": "impedance",<br/> "parameters": {<br/>   "k_stiffness": 0.45,<br/>   "b_damping": 0.12,<br/>   "max_torque_nm": 1.5<br/> }<br/>}

    MCU->>MCU: Update Flash Memory
    MCU->>MCU: Reboot / Ready for Next Session
```
