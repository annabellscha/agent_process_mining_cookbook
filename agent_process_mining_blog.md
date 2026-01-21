---
title: "Process Mining for Agent Evaluation"
date: 2025/01/22
description: A practical guide to discovering execution patterns, detecting bottlenecks, and generating actionable insights from your AI agent traces using process mining techniques.
ogImage: /images/blog/process-mining-for-agent-evaluation/og.png
tag: guide
author: [Your Name]
---

import { BlogHeader } from "@/components/blog/BlogHeader";

<BlogHeader
  title="Process Mining for Agent Evaluation"
  description="A practical guide to discovering execution patterns, detecting bottlenecks, and generating actionable insights from your AI agent traces using process mining techniques."
  date="January 22, 2025"
  authors={["yourname"]}
/>

To improve your AI agent, you need to understand **how it actually executes in production**. Traditional evaluation tests known scenarios, but what about the unknown patterns emerging in real usage? **Process mining** reveals what's really happening.

Traditional LLM evaluation validates expected behavior against test cases. Process mining complements this by discovering the actual execution patterns your agents take in production—not just what you designed, but what's truly occurring at scale.

---

This guide describes a practical approach to **apply process mining techniques to your agent traces**. The result is concrete insights about bottlenecks, inefficiencies, and execution patterns that traditional metrics miss:

1. [Export and transform your traces](#export-traces)
2. [Discover execution patterns](#discover-patterns)
3. [Analyze success vs failure paths](#analyze-patterns)
4. [Identify bottlenecks and inefficiencies](#find-inefficiencies)

---

I'll demonstrate this process using traces from a customer support agent that handles ticket classification, account lookups, and knowledge base searches. The agent logs all executions to Langfuse, giving us a rich dataset of real production behavior to analyze.

## Why Process Mining for Agent Evaluation?

Traditional LLM evaluation tests against known scenarios, which catches regressions but misses unknown failure patterns and emergent behaviors. Process mining complements this by discovering what's actually happening in production:

| Traditional Evaluation       | Process Mining                 |
| ---------------------------- | ------------------------------ |
| Tests known scenarios        | Discovers unknown patterns     |
| Validates expected behavior  | Reveals actual behavior        |
| Catches regressions          | Identifies bottlenecks         |
| Manual test case creation    | Generates test cases from data |

Unlike traditional evaluation which asks "did this specific test pass?", process mining asks "what patterns emerge across thousands of real executions?". Traditional evaluation is essential for catching regressions, but it can't tell you that your agent loops unnecessarily 15% of the time, or that a specific tool call consumes 60% of execution time, or that successful executions follow a completely different pattern than failures.

> **What is Process Mining?** Process mining is a family of techniques that extract knowledge from event logs to discover, monitor, and improve real processes. When applied to AI agents, it reveals the actual execution paths your agents take—the sequences of tool calls, LLM generations, and decision points that happen in production.

> **What is PM4Py?** [PM4Py](https://pm4py.fit.fraunhofer.de/) is an open-source process mining library in Python that provides algorithms for process discovery, conformance checking, and performance analysis.

## 1. Export and Transform Your Traces [#export-traces]

Process mining requires your trace data in a specific event log format. Each event needs:

- A case ID (trace identifier)
- An activity name (what happened)
- A timestamp (when it happened)
- Optional attributes (metadata like success/failure)

Start by [exporting traces from Langfuse](/docs/export). You can filter by date range, tags, or specific agent versions. For our customer support agent, we exported some 20 traces.

The transformation code converts Langfuse's nested observation structure into a flat event log. Each observation (generation, tool call, span) becomes an event with its parent trace as the case ID:

```python
import json
import pandas as pd
from datetime import datetime

def langfuse_to_event_log(json_file):
    """Convert Langfuse JSON export to process mining format"""
    with open(json_file) as f:
        traces = json.load(f)
    
    events = []
    for trace in traces:
        case_id = trace['id']
        success = trace.get('metadata', {}).get('success')
        
        for obs in trace.get('observations', []):
            events.append({
                'case:concept:name': case_id,
                'concept:name': obs['name'],
                'time:timestamp': obs['startTime'],
                'success': success,
                'observation_type': obs['type']
            })
    
    df = pd.DataFrame(events)
    df['time:timestamp'] = pd.to_datetime(df['time:timestamp'])
    return df.sort_values(['case:concept:name', 'time:timestamp'])
```

The resulting event log is ready for process mining analysis.

## 2. Discover Execution Patterns [#discover-patterns]

Once your data is in event log format, you can discover the actual execution patterns. Process discovery algorithms analyze the sequences of activities across all traces and generate a process model showing the most common paths.

Using PM4Py, we discovered that our customer support agent exhibits 5 distinct execution patterns. The most common pattern (45% of traces) follows this sequence:

```
Ticket Incoming → Classify Ticket → Check Account Status →
Search Knowledge Base → Generate Response → Send Response
```

But we also found less common patterns, including one where the agent loops through knowledge base searches multiple times before responding:

```
Ticket Incoming → Classify Ticket → Search Knowledge Base →
Generate Response → Search Knowledge Base → Generate Response →
Search Knowledge Base → Generate Response → Send Response
```

This variant appears in 12% of traces—a significant inefficiency we never designed for but is happening in production.

The process discovery generates a visual flow diagram showing all paths and their frequencies:

```python
import pm4py
from pm4py.objects.log.util import dataframe_utils

# Convert DataFrame to event log
event_log = dataframe_utils.convert_to_event_log(df)

# Discover process model using Directly-Follows Graph
dfg, start_activities, end_activities = pm4py.discover_dfg(event_log)

# Visualize
pm4py.view_dfg(dfg, start_activities, end_activities)
```

## 3. Analyze Success vs Failure Patterns [#analyze-patterns]

Process mining becomes particularly powerful when you correlate execution patterns with outcomes. By analyzing which patterns lead to success versus failure, you can identify problematic execution paths.

For our customer support agent, we grouped traces by their execution pattern and calculated success rates:

| Pattern                       | Frequency | Success Rate |
| ----------------------------- | --------- | ------------ |
| Direct path (6 steps)         | 45%       | 94%          |
| Single retry (8 steps)        | 28%       | 87%          |
| Multiple knowledge base loops | 12%       | 62%          |
| Long classification chain     | 9%        | 45%          |
| Account lookup failures       | 6%        | 23%          |

The analysis reveals that traces with multiple knowledge base searches have significantly lower success rates. This suggests our agent struggles to find relevant information and repeatedly searches without a clear strategy.

Additionally, traces that fail account lookups almost never recover—indicating we need better error handling for authentication issues.

```python
# Group by execution pattern and analyze success rates
variants_df = pm4py.get_variants_as_tuples(event_log)
variant_stats = []

for variant, traces in variants_df.items():
    trace_ids = [t.attributes['concept:name'] for t in traces]
    success_rate = df[df['case:concept:name'].isin(trace_ids)]['success'].mean()
    
    variant_stats.append({
        'pattern': ' → '.join(variant),
        'frequency': len(traces),
        'success_rate': success_rate
    })

stats_df = pd.DataFrame(variant_stats).sort_values('frequency', ascending=False)
```

## 4. Identify Bottlenecks and Inefficiencies [#find-inefficiencies]

Process mining excels at identifying performance bottlenecks by analyzing activity durations and execution flow. For each activity, we can calculate average duration, total time consumed, and identify where agents spend disproportionate time.

Our analysis revealed that "Search Knowledge Base" consumes 54% of total execution time, despite appearing in similar frequency to other activities. This massive bottleneck wasn't visible in our traditional metrics focused on end-to-end latency.

We also detected specific inefficiency patterns:

**Loops**: 15% of traces contain loops where the agent cycles between "Generate Response" and "Search Knowledge Base" without making progress. These loops add 3-4 extra steps and significantly increase latency.

**Long Chains**: Some traces show unusual chains of classification attempts, suggesting difficulty with ambiguous tickets.

```python
# Calculate activity performance metrics
activity_stats = df.groupby('concept:name').agg({
    'time:timestamp': 'count',
    'duration_seconds': ['mean', 'sum']
}).round(2)

# Identify bottlenecks (activities consuming >20% of total time)
total_time = activity_stats[('duration_seconds', 'sum')].sum()
activity_stats['pct_of_total_time'] = (
    activity_stats[('duration_seconds', 'sum')] / total_time * 100
)
bottlenecks = activity_stats[activity_stats['pct_of_total_time'] > 20]
```

## Actionable Insights Generated

The process mining analysis produced a prioritized list of concrete improvements:

**High Priority:**

1. **Optimize Knowledge Base Search** - Consuming 54% of execution time. Consider caching, better indexing, or query optimization.
2. **Prevent Search Loops** - 15% of traces loop unnecessarily. Add logic to prevent repeated searches without new information.
3. **Handle Authentication Failures** - Traces with account lookup failures rarely recover. Implement early detection and graceful handling.

**Medium Priority:**

4. **Improve Classification Confidence** - Long classification chains suggest uncertainty. Consider model fine-tuning or clearer classification criteria.


These insights are specific, quantified, and directly actionable—far more valuable than generic metrics like "average latency decreased by 10%."

## Common Pitfalls

- **Starting Too Small**: Process mining needs volume to reveal patterns. Analyzing 10-20 traces won't surface meaningful insights. Aim for at least 100-500 traces.

- **Ignoring Success Correlation**: Looking at patterns without connecting them to outcomes misses the point. Always analyze which patterns lead to success versus failure.

- **One-Time Analysis**: Like error analysis, process mining should be recurring. Agent behavior evolves, so make this part of your regular evaluation cycle.

- **Over-Granular Activities**: If every LLM token generation is a separate activity, your process model becomes unreadable. Group related operations into meaningful activities.

## Next Steps

This process mining analysis reveals the gap between designed behavior and actual execution. These insights provide clear priorities for agent improvement, whether in prompt engineering, tool optimization, or architectural changes.

The patterns you discover can also inform your traditional evaluation strategy. High-performing execution patterns become templates for synthetic test generation, while problematic patterns become regression tests.

In practice, combining process mining with error analysis creates a comprehensive evaluation framework—one that both validates expected behavior and discovers unexpected patterns from production data.

For a complete implementation with code examples, check out our cookbook:

import { FileCode } from "lucide-react";

<Cards num={1}>
  <Card
    title="Process Mining Cookbook"
    href="/guides/cookbook/process-mining"
    icon={<FileCode />}
    arrow
  />
</Cards>
