# Principle 1: Eliminate Idle Compute

**Problem**: Idle compute resources are a major source of waste in large organizations.  
- Forgotten dev/staging clusters keep running.  
- Containers stay active even when workloads finish.  

**DevOps++ Solution**:  
Integrate runtime activity checks into CI/CD workflows. Shut down or scale down compute automatically when workloads are inactive.  

**Benefits**:  
- Cost savings (reduce unused cloud spend).  
- Lower carbon emissions.  
- Less operational clutter.  

See example: [idle-compute-cleanup.yaml](../../examples/kubernetes/idle-compute-cleanup.yaml)
