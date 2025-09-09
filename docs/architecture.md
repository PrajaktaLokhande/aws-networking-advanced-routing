# Architecture

```mermaid
flowchart LR
  subgraph AWS["VPC 10.70.0.0/16"]
    subgraph PUB["Public 10.70.1.0/24"]
      BAST[(Bastion EC2)]
      RT_PUB[RT: 0.0.0.0/0 → IGW]
    end

    subgraph DMZ["DMZ 10.70.2.0/24"]
      FW[(Firewall EC2\neth0 in DMZ (public)\neth1 in Private)]
      RT_DMZ[RT: 0.0.0.0/0 → IGW]
    end

    subgraph PRIV["Private 10.70.3.0/24"]
      APP[(App EC2\nNo public IP)]
      RT_PRIV[RT: 0.0.0.0/0 → FW (eni-*)]
    end

    IGW[Internet Gateway]
  end

  BAST --- RT_PUB --- IGW
  FW --- RT_DMZ --- IGW
  APP --- RT_PRIV -.-> FW
```
