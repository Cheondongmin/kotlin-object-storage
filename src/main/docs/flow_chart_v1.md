## Detailed S3-like Object Storage Flowchart

```mermaid
flowchart TD
    subgraph Client
        A(Start)
        B(Create Bucket)
        C(Upload Object)
        D(Initiate Multipart Upload)
        E(Upload Part)
        F(Complete Multipart Upload)
        G(Get Object)
    end

    subgraph API_Layer
        LB("Load Balancer / API Gateway")
        APISvc("API Service")
        IAM["Auth Service (IAM)"]
    end

    subgraph Control_Plane
        MetaSvc("Metadata Service")
        ConfigSvc("Configuration Service")
    end

subgraph Data_Plane
Router(Request Router)
DataSvc(Data Service)
StorageNodes["Storage Nodes\n(Erasure Coding & Replication)"]
end

subgraph Responses
B_Resp(Bucket Created)
C_Resp(Upload Success)
D_Resp(UploadID)
E_Resp(Part ETag)
F_Resp(Complete Success)
G_Resp(Object Data)
end

A --> B
B --> LB --> APISvc --> IAM --> APISvc --> MetaSvc --> APISvc --> B_Resp

C --> LB --> APISvc --> IAM --> APISvc --> MetaSvc --> APISvc --> Router --> DataSvc --> StorageNodes --> DataSvc --> APISvc --> MetaSvc --> APISvc --> C_Resp

D --> LB --> APISvc --> IAM --> APISvc --> MetaSvc --> APISvc --> D_Resp

E --> LB --> APISvc --> IAM --> APISvc --> DataSvc --> StorageNodes --> DataSvc --> APISvc --> MetaSvc --> APISvc --> E_Resp

F --> LB --> APISvc --> IAM --> APISvc --> DataSvc --> StorageNodes --> DataSvc --> APISvc --> MetaSvc --> APISvc --> F_Resp

G --> LB --> APISvc --> IAM --> APISvc --> MetaSvc --> APISvc --> Router --> DataSvc --> StorageNodes --> DataSvc --> APISvc --> G_Resp
```
~~~~