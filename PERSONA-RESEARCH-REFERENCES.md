# Persona Research — References

Tanggal: 2026-08-25 · Klasifikasi sumber menurut keandalan:
1 = official docs / engineering posts · 2 = GitHub source/spec ·
3 = technical writeups maintainers · 4 = peer-reviewed paper ·
5 = community documentation · 6 = forum/reddit (praktik, bukan
arsitektur authoritative)

---

## Eksperimen internal (raw data persisten di repo)

- experiments/2026-08-25-persona-ablation/ — harness, kondisi exact,
  hasil 2 run. [internal F]
- experiments/2026-08-25-dga/ — DGA pre-registered, raw_runs.json,
  analyze_dga.py. [internal F]
- experiments/2026-08-25-identity-grounding/ — IGE raw + analisis.
  [internal F]

## Eksternal

### Official docs / specs

1. SillyTavern Docs — Prompts & Post-History Instructions.
   https://docs.sillytavern.app/usage/prompts [1][F]
2. SillyTavern Docs — Prompt Manager (urutan & depth injeksi).
   https://github.com/SillyTavern/SillyTavern-Docs/blob/main/Usage/Prompts/prompt-manager.md [1][F]
3. SillyTavern Docs — World Info / Lorebook.
   https://docs.sillytavern.app/usage/core-concepts/worldinfo/ [1][F]
4. Character Card V2 Specification.
   https://github.com/malfoyslastname/character-card-spec-v2 [2][F]
5. Kindroid memory docs. https://kindroid.ai/docs/article/memory/ [1][F]
6. Nomi wiki/beginner guide. https://wiki.nomi.ai/Nomi_101_A_Beginners_Guide [5][F]
7. Replika memory help center.
   https://help.replika.com/hc/en-us/articles/37208679176077 [1][F]
8. Anthropic Alignment — The Persona Selection Model (Feb 2026).
   https://alignment.anthropic.com/2026/psm [1][F]

### Research compiles / technical writeups

9. Lin-Guanguo/llm-memory-research — character-ai.research.md
   (kompilasi blog resmi C.AI + analisis Nathan Lambert).
   https://github.com/Lin-Guanguo/llm-memory-research/blob/main/character-ai.research.md [3][F/I]
10. Lin-Guanguo/llm-memory-research — personality-engineering.research.md
    (survey prompting vs fine-tuning vs activation engineering;
    termasuk praktik SillyTavern dan temuan kimjammer).
    https://github.com/Lin-Guanguo/llm-memory-research/blob/main/personality-engineering.research.md [3][F/CP]
11. KinthAI — Why Character.AI Forgets You (analisis arsitektur memory
    C.AI). https://blog.kinthai.ai/why-character-ai-forgets-you-persistent-memory-architecture [3][I]

### Peer-reviewed papers / preprints

12. Liu et al., "Lost in the Middle: How Language Models Use Long
    Contexts." TACL 2023. https://arxiv.org/abs/2307.03172 [4][F]
13. Laban et al., "LLMs Get Lost In Multi-Turn Conversation." 2025.
    https://arxiv.org/abs/2505.06120 [4][F]
14. "When 'A Helpful Assistant' Is Not Really Helpful: Personas in
    System Prompts Do Not Improve Performances." 2311.10054.
    https://arxiv.org/html/2311.10054v2 [4][F]
15. RoleMRC benchmark. https://arxiv.org/html/2502.11387v1 [4][F]
16. BIG5-CHAT, ACL 2025. https://aclanthology.org/2025.acl-long.999/ [4][F]
17. OpenCharacter. https://arxiv.org/abs/2501.15427 [4][F]
18. Persona-Aware Contrastive Learning. https://arxiv.org/html/2503.17662v1 [4][F]
19. PRISM routing. https://arxiv.org/html/2603.18507 [4][F]
20. LoRA Learns Less and Forgets Less. https://arxiv.org/abs/2405.09673 [4][F]
21. Consistently Simulating Human Personas with Multi-Turn RL.
    https://openreview.net/forum?id=A0T3piHiis [4][F]

### Community practice (bukan authoritative)

22. r/SillyTavernAI — Tips for character creation.
    https://www.reddit.com/r/SillyTavernAI/comments/179enor/ [6][CP]
23. r/SillyTavernAI — Persona vs Personality vs Description.
    https://www.reddit.com/r/SillyTavernAI/comments/1aq7k4l/ [6][CP]
24. r/SillyTavernAI — Characters change too fast (drift reports).
    https://www.redditmedia.com/r/SillyTavernAI/comments/1jhu1h5/ [6][CP]

## Catatan provenance

- Klaim internal C.AI tentang character training berasal dari analisis
  publik Nathan Lambert + pernyataan resmi terbatas; detail internal =
  [?] UNKNOWN.
- Perbandingan aplikasi companion (Nomi/Kindroid/Replika) memakai docs
  resmi masing-masing; artikel review pihak ketiga hanya untuk cross-
  check perilaku yang dilaporkan user.
- Semua klaim komunitas Reddit/forum diberi label [CP] dan TIDAK
  dipakai sebagai dasar keputusan arsitektur tanpa korelasi evidence
  lain.
