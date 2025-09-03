# Principle 3: Right-Size Memory With Runtime Feedback

**Problem**: Memory allocation is rarely correct on the first try.  
- Over-provisioning leads to waste.  
- Under-provisioning causes OOM (out-of-memory) crashes.  

**DevOps++ Solution**:  
Use runtime traces and historical data to **auto-update IaC files** with recommended memory limits. Integrate this into CI pipelines.  

**Benefits**:  
- Fewer crashes.  
- Reduced memory waste.  
- Continuous optimization of workloads.  
