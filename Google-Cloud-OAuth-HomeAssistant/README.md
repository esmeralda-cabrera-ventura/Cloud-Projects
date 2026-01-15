                                                         Home Assistant Google Cloud Integration

This is an end-to-end design and implementation of a Google Cloud–backed smart-home integration, connecting a Google Nest Thermostat to a locally hosted Home Assistant instance using Google’s official Smart Device Management (SDM) platform. I approached this as an infrastructure and identity integration exercise rather than a simple device pairing, intentionally treating it as a hands-on cloud architecture project that required careful attention to authentication, API enablement, event-driven messaging, and secure service-to-service communication.

From a cloud infrastructure perspective, the foundation of the solution was a dedicated Google Cloud project configured as the security, billing, and API boundary for the integration. Billing was explicitly enabled, not because of anticipated cost, but because Google requires an active billing account to provision and operate core infrastructure services such as Cloud Pub/Sub and the SDM API. Within this project, I enabled the Smart Device Management API, which serves as the authoritative interface through which third-party applications can access Nest device telemetry, state, and control functions. This API layer is the sole supported mechanism for external platforms like Home Assistant to interact with Nest hardware.

On top of the Cloud project, I designed and configured the OAuth 2.0 identity layer using Google’s Auth Platform. I created an external OAuth application and promoted it to “In production” status to support real user authorization flows. The OAuth client was defined as a web application, representing Home Assistant as a trusted relying party within Google’s identity system. A critical detail in this architecture was the precise configuration of authorized redirect URIs: because Home Assistant was running locally rather than through Home Assistant Cloud, the OAuth callback endpoint had to match the instance’s local network address exactly. Once the correct local callback URL was added, Google was able to complete the OAuth flow and securely return authorization tokens to Home Assistant. This step ultimately resolved the silent authentication failures that occur when identity and redirect boundaries do not align.

In parallel, I enrolled in Google’s Device Access program and created a Device Access Console project, which functions as the Nest-specific access control layer. This project establishes an explicit trust relationship between Nest devices and third-party applications. The Device Access project was linked to the previously created OAuth client and granted the required SDM service scope, forming a complete trust chain from Nest hardware, through Google’s device backend, into the Cloud project, and finally to Home Assistant. Architecturally, this separation cleanly enforces least privilege while keeping device authorization distinct from general cloud resource management.

To support real-time state updates rather than periodic polling, I integrated Google Cloud Pub/Sub into the solution. During the Home Assistant setup flow, a Pub/Sub topic and subscription were created within the Cloud project to receive Nest device events. This event-driven messaging layer allows Google to push thermostat state changes, such as temperature updates, HVAC mode changes, and system activity, directly to Home Assistant as they occur. In practice, this results in near-instant synchronization between the physical device and the Home Assistant dashboard while remaining well within Google’s free-tier usage limits for Pub/Sub.

The final architecture is a clean example of a modern cloud-to-local hybrid system: Nest devices communicate exclusively with Google’s backend; Google Cloud acts as the secure broker through OAuth, SDM APIs, and Pub/Sub; and Home Assistant operates as a locally hosted control plane that consumes cloud events and issues authenticated commands back through the same pipeline. The result is a fully functional, subscription-free Nest integration with real-time updates, strong identity guarantees, and minimal operational overhead, while also serving as a practical demonstration of my ability to design, troubleshoot, and implement identity-driven, event-based architectures on Google Cloud.

         ** END-TO-END ARCHITECTURAL WORKFLOW **
         
             ┌─────────────────────────┐
             │     Nest Thermostat     │
             └────────────┬────────────┘
                          │
                          ▼
                ┌─────────────────────────┐
                │    Google Nest / SDM    │
                │ Backend (Google-managed)│
                └────────────┬────────────┘
                             │
            ┌────────────────┴──────────────────┐
            │                                   │
            ▼                                   ▼
┌──────────────────────────┐          ┌──────────────────────────┐
│      SDM REST API        │          │    Cloud Pub/Sub Topic   │
│   (Commands \& Queries)  │          │   (Event-driven updates) │
└───────────── ────────────┘          └─────────────┬────────────┘
             │                                      │
             │                                      ▼
             │                         ┌──────────────────────────┐
             │                         │    Pub/Sub Subscription  │
             │                         └─────────────┬────────────┘
             │                                       │
             ▼                                       ▼
     ┌────────────────────────────────────────────────────┐
     │                   Home Assistant (Local)             
     │  - OAuth token storage                               
     │  - Consumes Pub/Sub events                           
     │  - Issues SDM control commands                       
     └────────────────────────────────────────────────────┘
                                ▲
                                │
                 OAuth 2.0 Authorization Code Flow
                                │
                 ┌──────────────────────────┐
                 │   Google OAuth / Identity  
                 │   Platform                 
                 └──────────────────────────┘

┌──────────────────────────┐
│   Device Access Project          
│ (SDM Scope \& Device Trust) 
└─────────────┬────────────┘
              ▼
        Authorizes SDM API

