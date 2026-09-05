# RAC-UAgent: Risk-Aware Action Selection for Uncertainty-Calibrated Tool-Using Language Model Agents

> **RAC-UAgent** stands for **Risk-Aware Control for Uncertainty-Calibrated Agents**.**Research status:** Theoretical framework and research proposal. The architecture and hypotheses are specified, but empirical validation is not yet complete.

Tool-using language-model agents can plan, retrieve information, call external tools, and execute multi-step workflows. However, a locally plausible action can still create a globally invalid or unsafe trajectory when the agent selects the wrong tool, supplies malformed arguments, trusts a misleading observation, or continues after uncertainty becomes decision-relevant.

This paper proposes **RAC-UAgent**, an **uncertainty-calibrated, risk-aware control architecture** for reliable multi-step agent execution. The framework separates planning, uncertainty estimation, verification, and action control. It allows an agent to choose among execution, retrieval, replanning, clarification, and human escalation instead of treating every generated action as equally trustworthy.

## Core idea

RAC-UAgent evaluates candidate actions before execution using several signals:

1. **Reasoning-path agreement**, measuring whether independent reasoning attempts support the same action.

1. **Tool-result consistency**, checking whether observations agree with expected outcomes and prior state.

1. **Retrieval confidence**, estimating the quality and agreement of supporting evidence.

1. **Action validity**, checking logical, syntactic, semantic, and policy constraints.

1. **Risk and cost**, accounting for potential harm, reversibility, latency, verification cost, and remaining trajectory horizon.

The controller then selects the action with the best risk-adjusted utility. Depending on the estimated risk, it may execute the action, verify it, retrieve more information, ask the user for clarification, replan, or escalate to a human.

## Architecture

```
                         Task goal and history
                                  |
                                  v
                              Planner
                                  |
                         Candidate action set
                                  |
          +-----------------------+-----------------------+
          |                       |                       |
          v                       v                       v
   Uncertainty estimator       Verifier              Risk model
   - path agreement            - logical             - harm
   - result consistency        - syntactic           - reversibility
   - retrieval confidence      - semantic            - horizon
   - action validity           - policy              - cost
          |                       |                       |
          +-----------------------+-----------------------+
                                  |
                                  v
                        Risk-aware controller
                                  |
       +------------+-------------+-------------+------------+
       |            |                           |            |
       v            v                           v            v
    Execute      Retrieve                   Replan      Clarify / escalate
       |            |                           |            |
       +------------+-------------+-------------+------------+
                                  |
                                  v
                         Observation and outcome
                                  |
                                  v
                         Calibration feedback
```

## Research contributions

The paper makes four intended contributions. First, it defines an agent architecture that explicitly separates planning, uncertainty estimation, verification, and risk-sensitive control. Second, it proposes a composite action-risk estimator that combines agreement, retrieval, tool-output, and validity signals. Third, it formalizes trajectory success, selective abstention, cost-sensitive utility, tail risk, and calibration error accumulation. Fourth, it specifies a reproducible evaluation protocol with baselines, ablations, adversarial tool conditions, distribution-shift tests, and repeated-trial reliability measurements.

These contributions are presented as **research hypotheses and implementable design proposals**, not as claims that the complete system has already been validated.

## Formal problem

Let the agent history at time $$t$$ be $$h_t$$, and let $$a$$ be a candidate action. The conditional probability that the action fails is defined as

$$
p_t(a) = P(Y_t(a)=0 \mid h_t,a),
$$

where failure may include an incorrect action, invalid arguments, policy violation, unsafe execution, or a result that does not satisfy the current task state.

The agent produces an estimated failure probability $$\hat p_t(a)$$. The central calibration requirement is that predicted risk should correspond to observed failure frequency. This distinction is important because language-model confidence is not automatically a calibrated probability of tool success.

For a trajectory of length $$T$$, the conditional-independence approximation gives

$$
P(\text{success}) = \prod_{t=1}^{T}(1-p_t).
$$

This motivates horizon-aware control: even moderate per-action risk can become significant over a long trajectory, particularly when failures are irreversible or difficult to recover from.

## Risk-aware action selection

RAC-UAgent evaluates each candidate action using a calibrated risk estimate, hard safety constraints, expected utility, cost, and tail-risk penalties. A simplified form of the objective is

$$
J(a) = U(a) - \lambda R(a) - \mu C(a),
$$

where $$U(a)$$ is expected utility, $$R(a)$$ includes estimated failure and tail risk, and $$C(a)$$ includes verification, retrieval, latency, and interaction costs.

Hard constraints are applied before utility optimization. An action that is unauthorized, syntactically invalid, semantically incompatible with the task, or otherwise unsafe should not become acceptable merely because its expected reward is high.

The controller may perform the following actions:

| Decision | When it is appropriate |
| --- | --- |
| Execute | Risk is below the execution threshold and verification passes. |
| Retrieve | Additional information has positive expected value. |
| Verify | A low-cost check can materially reduce action risk. |
| Replan | Current assumptions or observations are inconsistent. |
| Clarify | User intent is ambiguous and clarification is cheaper than failure. |
| Escalate | Risk, harm, or uncertainty exceeds the autonomous-operation boundary. |

## Threat model

The framework considers five major failure sources:

- Hallucinated plans or unsupported action arguments.

- Syntactically invalid or schema-incompatible tool calls.

- Semantically valid calls that do not satisfy the intended goal.

- Stale, misleading, corrupted, or adversarial tool observations.

- Compounding state errors in which an early failure invalidates later assumptions.

The design does not assume that external tools are always truthful. Instead, it treats tool observations as evidence that must be checked against expected outcomes, state, and other signals.

## Theoretical results

The paper provides several propositions that motivate the architecture:

- **Calibration error accumulation:** bounded per-step calibration errors can accumulate over long trajectories, so action-level calibration matters in addition to final-answer quality.

- **Verification benefit:** verification improves trajectory success when its reduction in failure risk outweighs its cost and false-rejection rate.

- **Log-risk accumulation:** the logarithm of trajectory success decomposes into a sum of per-step log risks, motivating horizon-aware control.

- **Clarification value:** clarification is preferable when its reduction in ambiguity, risk, and downstream error exceeds the interaction cost.

These results provide sufficient conditions and analytical intuition. They do not guarantee that a practical uncertainty estimator will satisfy the assumptions under distribution shift.

## Proposed evaluation protocol

RAC-UAgent is designed to be evaluated as an **action-selection controller**, not merely as a prompting technique. The recommended protocol compares otherwise matched agents that differ in their planning, verification, calibration, and control policies.

### Research questions

| Question | Evaluation target |
| --- | --- |
| RQ1: Calibration | Does the estimator reduce action-level ECE and Brier score? |
| RQ2: Reliability | Does risk-aware selection improve task success and pass@k while reducing invalid or catastrophic actions? |
| RQ3: Efficiency | Does adaptive verification improve the cost–reliability frontier compared with verifying every action? |
| RQ4: Robustness | Does the method remain reliable under misleading observations, schema changes, partial failures, and distribution shift? |
| RQ5: Human interaction | Does uncertainty-triggered clarification reduce failure without creating excessive unnecessary questions? |

### Recommended benchmarks

The evaluation plan considers interactive web, tool-use, conversational API, software-engineering, and database environments, including WebArena, BrowserGym, ToolBench, τ-bench, SWE-bench, and controlled API/tool environments.

### Metrics

A meaningful evaluation should report more than aggregate task success:

- Task success with confidence intervals.

- Action-level expected calibration error and Brier score.

- Reliability diagrams and risk–coverage curves.

- Repeated-trial success and pass@k by trajectory horizon.

- Invalid-call, unsafe-action, and catastrophic-action rates.

- Verification precision, recall, and unnecessary-intervention rate.

- Clarification and escalation frequency.

- Steps, tokens, latency, tool calls, and cost per successful task.

- Error categories such as wrong tool, malformed argument, stale retrieval, verifier miss, state-transition failure, and unsafe execution.

### Leakage controls

Tasks should be split by task template and tool family rather than only by random instance. Calibration labels must be separated from planner tuning. Repository versions, browser versions, retrieval corpora, tool schemas, and external knowledge sources should be pinned and recorded. These controls are necessary to distinguish genuine reliability from benchmark-specific memorization or leakage.

## Current status and limitations

The RAC-UAgent repository currently contains the paper and its proposed methodology. It does **not** claim completed empirical validation of the full architecture. The evaluation tables in the paper specify recommended experiments and reporting requirements; numerical results are intentionally left for future work.

Important limitations include the following:

1. Calibration may degrade under deployment distribution shift.

1. Agreement between reasoning paths or verifiers may be correlated and therefore overconfident.

1. Tool-induced feedback can violate assumptions such as exchangeability.

1. A conservative controller may improve safety by escalating too often, so intervention cost must be measured.

1. The theoretical results rely on stated assumptions and are not universal guarantees.

1. Benchmark success alone cannot establish real-world safety or reliability.

## Roadmap

The planned implementation path is:

- Define a common structured action schema and tool adapter interface.

- Implement logical, syntactic, semantic, and policy verification layers.

- Build the multi-signal uncertainty estimator.

- Create calibration and held-out test splits by task template and tool family.

- Implement selective execution, retrieval, clarification, replanning, and escalation.

- Evaluate calibration, risk coverage, trajectory success, and tail failure rates.

- Add adversarial observations, schema perturbations, stale retrieval, and distribution-shift tests.

- Compare against ReAct, self-consistency, verifier-only, retrieval-augmented, and uncalibrated controllers.

## Relationship to related projects

This research direction is intended to connect naturally with agent orchestration and safety tooling. It can provide the theoretical control layer for a research system such as **Athena** and the uncertainty, verification, and escalation logic for a safety wrapper such as **Panopticon**.

## Citation

```
@article{varma2026riskaware,
  title   = {Risk-Aware Action Selection for Uncertainty-Calibrated Tool-Using Language Model Agents},
  author  = {Varma, Akhilesh},
  year    = {2026},
  month   = {September},
  note    = {Research proposal and theoretical framework}
}
```

## License

Add the license that applies to the paper and any accompanying implementation. If you have not selected one yet, choose a license before publishing source code or reusable experiments.

## Disclaimer

This work is a research proposal and is not a guarantee of agent safety, correctness, or reliable performance in deployed environments. It should not be used as the sole basis for granting an autonomous system access to sensitive tools, financial systems, production infrastructure, personal data, or irreversible actions.
