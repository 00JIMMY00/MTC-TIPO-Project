# Data Flow: AI-Driven Elbow Splint System

This document outlines the data flow from sensor collection to motor adjustment, involving daily cloud synchronization for personalized treatment adjustments.

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

## Data Flow Description

1.  **Data Acquisition**:
    *   **Source**: Patient's arm.
    *   **Sensors**: EMG (muscle activity), IMU (arm orientation/acceleration), Encoder (joint angle).
    *   **Destination**: Microcontroller receives this data in real-time.

2.  **Local Control & Buffering**:
    *   The Microcontroller runs a base control loop to assist the patient immediately.
    *   Simultaneously, it logs session data (performance metrics, range of motion, spasticity events) locally.

3.  **Daily Cloud Sync**:
    *   Once a day (or post-session), the Microcontroller packages the collected data and pushes it to the Cloud.

4.  **Cloud Analysis & Model Refinement**:
    *   The Cloud processes the historical data to evaluate patient progress.
    *   It determines if the splint needs to be "harder" (more rigid for support) or "softer" (more compliant for comfort/safety).
    *   It generates a new configuration profile (JSON).

5.  **Parameter Update & Motor Adjustment**:
    *   The Cloud sends the JSON body back to the Microcontroller.
    *   The Microcontroller parses the JSON and updates its internal PID/Impedance control parameters.
    *   **Result**: The Stepper Motors now behave differently (e.g., increased resistance or gentler assistance) in the next session.
