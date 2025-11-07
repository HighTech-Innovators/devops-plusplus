# P2 – Challenge CPU Load Without Business Context

## 1. Definition

### Key Concepts
- CPU load from code not linked to real user actions or business processes should be challenged to prove its value or refactored for meaningful outcomes.  
- Developers should be challenged when making architectural decisions that impact CPU load.  
- Business should also be challenged by developers — alignment between business and technical goals is crucial.  
- An early conversation is needed to align resource specifications (CPU, GPU) to business goals.  
- Business and Development often perceive software differently (e.g., **Software as an Asset vs. Liability**).  
- Currently, there is limited overlap between business and technical domains, which makes it hard to get complete answers.  

---

## 2. Why It Matters

### Key Benefits
- Reduces unnecessary compute usage, which saves infrastructure costs and protects profit margins.  
- Prevents slowdown of systems for real users and key processes.  
- Keeps architecture clean, scalable, and maintainable.  

### Deeper Insights
- Unnecessary compute consumes budget without contributing to value creation.  
- Compute resources directly relate to cost and emissions — avoiding waste supports sustainability goals (e.g., lower CO₂ emissions).  
- Early focus on efficient compute is essential — once scaling begins, it becomes much harder to correct inefficiencies.  
- During startup phases, minimal focus on compute efficiency is acceptable, but this must change as products mature and scale.  

---

## 3. Examples

### Inefficient Patterns
- A cloud app recalculating data every minute even when nothing changes — wastes CPU and slows performance.  
- Constant transcoding of the same videos by a video platform — consumes compute and reduces profit margins.  
- Unused test environments running monthly in the cloud — adds cost without delivering customer value.  
- Overpowered CPUs or GPUs idling or overused compared to process needs (e.g., web servers, routers).  
- “Keeping GPUs warm” for AI workloads without actual requests — leads to inefficiency and unnecessary costs.  
- Real-world example: **€30,000 per month cloud spend** for ~100 users, primarily absorbed by unnecessary compute.  

---

## 4. Solution

### Core Strategies
- Conduct an **absolute assessment** of required compute for each workload.  
- Understand how to use CPUs and GPUs as efficiently as possible (e.g., leverage special hardware features).  
- Use the **right chip for the job** — avoid both overkill and inefficiency.  
- Do not upgrade blindly; balance between maintaining older hardware and overconsuming new resources.  
- Measure CPU load changes after each feature release to identify and correct inefficiencies.  
- Establish **internal SLAs** between development and business to ensure accountability for compute consumption and optimization.  

---

## Summary
The **Challenge CPU Load Without Business Context** principle emphasizes aligning technical decisions with business value by questioning unnecessary compute usage. It aims to reduce costs, improve efficiency, and promote environmental responsibility.  

For now, we will continue exploring this principle from **three perspectives: Frank, Luuk, and Wilco’s**, and **revisit it in 1.5 weeks** for a deeper dive into **Principle 2**.
