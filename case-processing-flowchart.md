# Case Processing Technical Design Flowchart

This flowchart shows the complete technical design for case processing from ECC to Salesforce.

## Process Flow

```mermaid
flowchart TD
    A[ECC: Split Case into Multiple Payloads] --> B[ECC: Publish Payloads to AMQ]
    B --> C[Mule: Listen & Read Payloads from AMQ]
    C --> D[Mule: Call SF API #1<br/>Create Staging Records<br/>Attach Payloads as Content Version]
    D --> E[Mule: Call SF API #2<br/>Trigger Queueable #1]
    
    E --> F[SF Queueable #1:<br/>Query Content Version by Split ID<br/>Retrieve Blob Payload]
    F --> F1{Retry Logic<br/>Attempt < 3?}
    F1 -->|Success| G[SF Queueable #2:<br/>Split Blob into onlyLines & onlySublines]
    F1 -->|Failed & Retry| F2[Wait with Exponential Backoff]
    F2 --> F
    F1 -->|Failed & Max Retries| F3[Flag for Manual Intervention]
    
    G --> G1{Has Sublines?}
    G1 -->|Yes| G2[Update Staging Record:<br/>subLines present = TRUE]
    G1 -->|No| G3[Keep subLines present = FALSE]
    G2 --> G4[Upsert Amended Lines<br/>Store onlySublines as Content Version]
    G3 --> G4
    G4 --> G5{Retry Logic<br/>Attempt < 3?}
    G5 -->|Success| H[SF Queueable #3:<br/>Process onlyLines]
    G5 -->|Failed & Retry| G6[Wait with Exponential Backoff]
    G6 --> G
    G5 -->|Failed & Max Retries| G7[Flag for Manual Intervention]
    
    H --> H1{Can Process Lines<br/>in Single Iteration?}
    H1 -->|Yes| H2[Upsert Lines<br/>Bypass Line Triggers]
    H1 -->|No| H3[Process Lines in Batches<br/>Iteratively Call Queueable #3]
    H3 --> H2
    H2 --> H4{Retry Logic<br/>Attempt < 3?}
    H4 -->|Failed & Retry| H5[Wait with Exponential Backoff]
    H5 --> H
    H4 -->|Failed & Max Retries| H6[Flag for Manual Intervention]
    H4 -->|Success| H7{Sublines Present<br/>Across All Lines?}
    
    H7 -->|Yes| I[SF Queueable #4:<br/>Process Sublines]
    H7 -->|No| H8{Next Split ID<br/>Exists?}
    H8 -->|Yes| H9[Call Queueable #1<br/>with Next Split ID]
    H9 --> F
    H8 -->|No| J[Line Status Management<br/>Queueable]
    
    I --> I1{Can Process Sublines<br/>in Single Iteration?}
    I1 -->|Yes| I2[Query Content Version for onlySublines<br/>Prepare Cancellation & Reversal Sublines]
    I1 -->|No| I3[Process Sublines in Batches<br/>Iteratively Call Queueable #4]
    I3 --> I2
    I2 --> I4[Upsert Sublines<br/>Do NOT Bypass Triggers]
    I4 --> I5[Call EH Cancellation API<br/>for Cancelled/Reversal Sublines]
    I5 --> I6{Retry Logic<br/>Attempt < 3?}
    I6 -->|Failed & Retry| I7[Wait with Exponential Backoff]
    I7 --> I
    I6 -->|Failed & Max Retries| I8[Flag for Manual Intervention]
    I6 -->|Success| I9{Is Last Split ID?}
    
    I9 -->|Yes| K[Subline Status Management<br/>Queueable]
    I9 -->|No| I10[Call Queueable #1<br/>with Next Split ID]
    I10 --> F
    
    J --> J1[Execute Existing Line<br/>Status Management Logic]
    J1 --> J2[Mark Current Split<br/>as Completed]
    J2 --> J3{All Splits for Case<br/>Completed?}
    J3 -->|No| J4[End Processing<br/>Wait for Other Splits]
    J3 -->|Yes| J5{Any Failed Splits<br/>for this Case?}
    J5 -->|Yes| J6[Flag Case for<br/>Manual Intervention<br/>Do NOT call API #3]
    J5 -->|No| J7[Call API #3<br/>Notify ECC Case Completion]
    J7 --> J8{API #3 Call<br/>Successful?}
    J8 -->|No & Retry < 3| J9[Retry API #3<br/>with Backoff]
    J9 --> J7
    J8 -->|No & Max Retries| J10[Flag for Manual<br/>API #3 Intervention]
    J8 -->|Yes| J11[Update Case Status:<br/>ECC_Notified]
    
    K --> K1[Execute Existing Subline<br/>Status Management Logic]
    K1 --> K2[Mark Current Split<br/>as Completed]
    K2 --> K3{All Splits for Case<br/>Completed?}
    K3 -->|No| K4[End Processing<br/>Wait for Other Splits]
    K3 -->|Yes| K5{Any Failed Splits<br/>for this Case?}
    K5 -->|Yes| K6[Flag Case for<br/>Manual Intervention<br/>Do NOT call API #3]
    K5 -->|No| K7[Call API #3<br/>Notify ECC Case Completion]
    K7 --> K8{API #3 Call<br/>Successful?}
    K8 -->|No & Retry < 3| K9[Retry API #3<br/>with Backoff]
    K9 --> K7
    K8 -->|No & Max Retries| K10[Flag for Manual<br/>API #3 Intervention]
    K8 -->|Yes| K11[Update Case Status:<br/>ECC_Notified]
    
    %% Manual Intervention Paths
    F3 --> M1[Manual Intervention<br/>Dashboard]
    G7 --> M1
    H6 --> M1
    I8 --> M1
    J6 --> M1
    J10 --> M1
    K6 --> M1
    K10 --> M1
    
    M1 --> M2[Production Support<br/>Manual Retry/Resolution]
    M2 --> M3{Manual Action}
    M3 -->|Retry from Queueable #1| F
    M3 -->|Retry from Queueable #2| G
    M3 -->|Retry from Queueable #3| H
    M3 -->|Retry from Queueable #4| I
    M3 -->|Manual API #3 Call| M4[Manual API #3 Trigger]
    M4 --> J11
    
    %% End States
    J4 --> END1[End: Waiting for Other Splits]
    J11 --> END2[End: Case Successfully Completed<br/>ECC Notified]
    K4 --> END1
    K11 --> END2
    
    %% Styling
    classDef eccNode fill:#e1f5fe
    classDef muleNode fill:#f3e5f5
    classDef sfNode fill:#e8f5e8
    classDef decisionNode fill:#fff3e0
    classDef errorNode fill:#ffebee
    classDef endNode fill:#f1f8e9
    
    class A,B eccNode
    class C,D,E muleNode
    class F,G,H,I,J,K sfNode
    class F1,G1,G5,H1,H4,H7,H8,I1,I6,I9,J3,J5,J8,K3,K5,K8,M3 decisionNode
    class F3,G7,H6,I8,J6,J10,K6,K10,M1,M2 errorNode
    class END1,END2 endNode
