# Proposal Template

This template is for recording changes that are worth keeping as long-term design history. It is usually not for small fixes, but for things like:

- architectural adjustments
- core interface changes
- component boundary changes
- data model evolution
- rollout and migration strategy

A good proposal is not supposed to be long. It should let future readers answer three questions quickly:

1. Why was this change needed at the time?
2. Why was this solution chosen in the end?
3. What new constraints and follow-up work did it introduce?

## 1. Background

Describe the current problem, business context, trigger, and existing pain points.

Recommended questions:

- Where does the problem appear?
- Why must it be handled now?
- What happens if we do not handle it?

## 2. Goals

State clearly what this proposal is meant to solve.

- Goal 1:
- Goal 2:
- Goal 3:

## 3. Non-goals

State clearly what this proposal does not solve, so the scope does not keep expanding.

- Non-goal 1:
- Non-goal 2:

## 4. Current state and constraints

Describe the current implementation, organizational constraints, compatibility constraints, and external dependency limits.

- Technical status:
- Dependency status:
- Release constraints:
- Compatibility constraints:

## 5. Design

### 5.1 Overall approach

Use one or two paragraphs to explain the core idea.

### 5.2 Architecture or flow diagram

Prefer adding a Mermaid diagram to show key modules and data flow.

### 5.3 Key changes

- Module A:
- Module B:
- Module C:

### 5.4 Data and interface changes

- New interfaces:
- Field changes:
- Compatibility impact:
- Migration plan:

### 5.5 Error handling and degradation

- Possible failure points:
- Failure handling:
- Degradation strategy:

## 6. Observability

Do not skip this section. Any proposal entering the main path should answer:

- Which new logs are needed?
- Which metrics are needed?
- Is trace correlation required?
- How do we know the solution is working correctly?

## 7. Security and compliance

- Does it introduce new secrets or permissions?
- Does it expand the external access boundary?
- Does it involve sensitive data handling?

## 8. Test plan

- Unit tests:
- Integration tests:
- Regression tests:
- Benchmark or pressure tests:

## 9. Release plan

- Will there be phased rollout?
- How will rollback work?
- What are the milestones for each stage?

## 10. Risk assessment

- Risk 1:
- Risk 2:
- Mitigation for each:

## 11. Alternatives considered

Describe what other options were considered and why they were not chosen.

## 12. Open questions

- [ ] Question 1
- [ ] Question 2

## 13. Decision record

- Decision:
- Participants:
- Date:
