# Principle 6: Prune Data at the Source in Microservices

**Problem**: Over-communication in microservices causes network bloat.  
- Oversized payloads.  
- Unused fields clogging data streams.  

**DevOps++ Solution**:  
- Use telemetry to find unused fields.  
- Enforce payload size limits.  
- Promote **contract-first development** for leaner APIs.  

**Benefits**:  
- Reduced costs and bandwidth.  
- Faster communication between services.  
- Improved scalability of distributed systems.  
