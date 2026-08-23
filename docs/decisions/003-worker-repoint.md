# ADR 003: nontrivial work moves to a tabehodai-served worker

Date: 2026-08-23. Status: accepted, blocked on evidence.

Eric's delegation rule sends nontrivial implementation to Codex
gpt-5.6-sol at high effort. The target state is "all nontrivial work is
done by a tabehodai-served model." Prerequisite: management-plane#756, which
adds a qwen3.8-27b ladder rung and evaluates it against qwen3.6-27b on
real delegated briefs. The routing table changes only after that eval
supports it.
