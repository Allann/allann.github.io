# The Pendulum — Progress Report I

### What changed after the first swing: agents, harnesses, and the new scarcity

*An unpublished working companion to [The Pendulum](./the-pendulum-whitepaper.md). Written from public observations. Draft for review.*

---

## About this document

This is not a revision of *The Pendulum*, and it is not an attempt to rescue it from the world changing underneath it. The argument of that paper still stands where I put it: an agent does not make the uncertainty of building the right thing disappear merely by making it easier to produce a thing quickly.

But the world has moved quickly enough that I do not think the original formulation is quite sufficient on its own anymore.

When I began writing, the danger I could see most clearly was a familiar one. An organisation, newly impressed by what an agent could build, would decide that the answer was to specify more completely at the front: write the AI-ready prompt, get it approved, estimate it, and then hand it over the boundary to execution. The prompt was the specification; the approval was the gate; the agent supplied a new reason to believe that this time the gate might hold.

That pattern has not gone away. I still see it, and I still think it is a mistake. But a second pattern is now becoming visible in public. The most serious practitioners are no longer talking mainly about the prompt. They are talking about the environment around it: the codebase the agent can navigate, the decisions it can find, the checks it can run, the UI it can exercise, the logs and traces it can inspect, the permissions it has, and the evidence it can produce before anyone has to decide whether to trust it.

That is a meaningful development, because it asks a better question than “how do we tell the agent exactly what to do?” It asks: *what sort of system would let us discover, constrain, test, and correct work at the speed the agent now makes possible?*

I am calling that system the **harness**. I do not mean a product feature, or a vendor term, or a new methodology to put on a slide. I mean the surrounding conditions that make a generated change either a plausible surprise or a thing the team can actually stand behind. The repository knowledge. The interfaces. The tests. The deployment boundaries. The operational evidence. The small pieces of accumulated judgment that let someone, human or agent, tell the difference between “it ran” and “it is right enough to keep.”

The original paper argued that the unit of work could not honestly be the signed-off prompt. This progress report makes a stronger version of that claim: **the unit of AI-native engineering is not the prompt. It is the harness in which the prompt operates.**

The observations here are public. They come from research papers, vendor accounts, technical reports, and the increasingly candid writing of organisations learning how difficult this is. They are not proof that a particular local programme will succeed or fail. They are a second field of weather reports. As with the original paper, the aim is to notice the pattern before it becomes a doctrine, and to keep the counter-arguments close enough that I cannot quietly edit them out of my own view.

---

## Where this leaves the original argument

There is a sentence at the centre of *The Pendulum* that I would now qualify, not abandon. I wrote that the agent had changed the cost of writing code, but had not changed discovery or comprehension. That was a useful correction to the claim that implementation speed had somehow made product uncertainty vanish. It remains useful in that sense.

But it is too neat if read literally.

Agents do now help people understand things. They map unfamiliar codebases, follow call paths, explain modules, reproduce defects, compare possible designs, generate small experiments and in environments that allow it, inspect the behaviour of a running system through its UI, logs, metrics, and traces. That does not make them reliable knowers. It does not mean they understand a domain in the human sense, or know what consequence matters most, or carry the responsibility for a decision. But it does mean that discovery and comprehension can be accelerated, sometimes dramatically, when the surrounding system gives the agent something real to work with.

The bottleneck is not simply moving from implementation to specification. Nor is it sitting untouched outside the pipe, exactly where it always was. The scarce thing is becoming **trusted attention**: the time and judgment needed to decide what matters, decide what evidence is adequate, notice when the available evidence is misleading, and stay connected to the people and consequences on the other side of the software.

Code is becoming abundant. Plausible explanations are becoming abundant. Candidate solutions are becoming abundant. What is not abundant is the attention required to sort, test, connect, and take responsibility for them.

That is the observation this document is trying to catch.

---

## 1. The new constraint appears at the back of the loop

> **Section brief.** Revisit the original paper’s warning about verification. The current public evidence suggests that the first constraint exposed by higher agent throughput is not necessarily specification but the human and organisational capacity to assess, integrate, and own more change. The important distinction is between producing more output and producing more evidence.

The first time I read a serious account of an agent-first engineering environment, the thing that stayed with me was not the claim about how much code the agents could write. By now, we have all heard some version of that claim. It was the admission of where the constraint turned up next.

OpenAI’s account of its own experiment describes a team discovering that, as agent-produced code increased, human QA capacity became the fixed constraint. Their response was not simply to ask people to review faster. They made the application more legible to the agent: each worktree could run the application; the agent could drive the UI; it could inspect logs, metrics, and traces; it could gather evidence about its own change in an environment closer to the one it would actually inhabit. [OpenAI, “Harness engineering: leveraging Codex in an agent-first world”](https://openai.com/index/harness-engineering/)

I do not take a vendor’s account of its own work as proof that everyone should work this way. Nor should anyone. It is a particular organisation, unusually well equipped, describing a greenfield experiment in which it has every incentive to understand these tools deeply. But the mechanism is worth noticing because it is a recognisable one: once production is cheap, judgment at the back of the loop becomes visible as a scarce resource.

This is where the original paper needs extending. I argued there that approval of intent cannot become the only definition of done. That remains true. But “have a human read every retained line” is not, on its own, a durable answer to a system capable of producing far more retained lines than a team can patiently reconstruct in its head. If human reading is the only safety mechanism, then the faster producer wins eventually.

The better question is not whether a human has touched the output. It is: *what would count as evidence that this change deserves to be kept?* Some of that evidence will be human judgment. Some will be a test, a contract, a type, a migration rehearsal, a performance budget, a permission boundary, or a staged release. Some will be a person watching a real user try to do the thing the team thought it understood. The point is not to make everything mechanical. It is to stop spending the rarest kind of attention on work a good system could have made cheap to check.

The agent does not remove the need for verification. It makes the absence of a designed verification system harder to hide.

---

## 2. The prompt was never enough; now we can say what belongs around it

> **Section brief.** Distinguish the healthy response to agent context from the return of the frozen mega-spec. Public practice is converging on repository-local, structured, revisable, and mechanically checked knowledge, not one complete prompt.

There is a temptation, whenever an agent does something surprising, to respond by writing it a longer instruction. The instinct is understandable. If the result was wrong, perhaps the missing sentence was the cause. If the agent did not know the business rule, perhaps the answer is to add that rule to the master prompt. Before long, a document intended to make the next task safer has become the thing nobody reads, nobody trusts, and nobody is willing to delete because every line might be someone’s scar tissue.

That is not merely a hypothetical failure mode. OpenAI’s public account says it tried the single enormous `AGENTS.md` approach and found the predictable result: context was crowded out, everything presented itself as important, the document became stale, and it was difficult to verify whether it reflected reality. Their described alternative is modest in a way I find encouraging: a short map, pointers to deeper repository knowledge, versioned plans and decision records, and checks intended to expose documentation drift. [OpenAI, “Harness engineering”](https://openai.com/index/harness-engineering/)

This is not Waterfall with a better filing system. At least, it is not if it stays alive.

The distinction I drew in the original paper still does the work here. The question is whether the artefact is held as a hypothesis or as a contract. But there is now a further test: can the artefact be found, checked, corrected, and connected to the system it claims to describe? A business rule hidden in a former employee’s memory is not shared knowledge. Neither is a beautiful design document in a drive the agent cannot see and the next engineer will not think to search. Knowledge becomes useful only when the work can encounter it at the point it matters.

I am wary of making repositories sound like the new source of all truth. They can become graveyards too. A stale markdown file checked into Git is still stale. But the public accounts point toward a useful direction: context should be layered rather than encyclopaedic; close to the work rather than ceremonial; versioned rather than oral; and, where it can be, checked against reality rather than maintained by faith.

That gives the idea of Knowledge Debt a second form. The original was personal and team-level: code arrives faster than understanding, leaving a widening gap between what the system contains and what the people responsible for it comprehend. The second form is organisational: vital judgment exists, but is not recoverable. It lives in a chat thread, a departed colleague, a meeting no one wrote down, or a document the real work cannot reach. The agent did not create that debt. It makes its interest payments harder to ignore.

---

## 3. Verifiability belongs on the map

> **Section brief.** Extend the original paper’s risk / novelty / complexity space. A task’s safety under delegation also depends on whether correctness can be established cheaply and reliably before harm is difficult to reverse.

The original paper made the case that a universal agent playbook was wrong because work sits in different places on a map of risk, novelty, and complexity. I would keep that map. I would add one more axis: **verifiability**.

By verifiability, I mean the cost and reliability of answering a deceptively plain question: *how will we know whether this is good enough before its consequences become difficult to undo?*

This is not the same as risk. A change can be high risk and highly verifiable: perhaps there is a precise simulation, a strong independent oracle, or a carefully controlled release path. It can also be small and apparently ordinary but difficult to verify: a change to wording in a regulated flow, an integration that fails only under a rare production condition, an access-control rule whose failure is invisible until it matters. Low novelty does not automatically mean safe to delegate. Familiar work inside an opaque system can be more dangerous than novel work inside a well-instrumented one.

Once that axis is visible, the practical question changes. Not “are agents allowed to do this?”, because that question flatters us into thinking the tool is the whole decision. Instead: *what evidence exists; who can judge it; what happens if it is wrong; and can we reverse it?*

OpenAI’s more ordinary Codex guidance, separate from its agent-first experiment, points in the same direction without making a philosophical argument of it. It recommends scoped tasks, planning before large changes, configured environments, reliable tests, and iterative improvement to the environment rather than an attempt to write one perfect instruction. [OpenAI, “How OpenAI uses Codex”](https://openai.com/business/guides-and-resources/how-openai-uses-codex/)

That is what a healthy harness does. It does not claim to eliminate judgment. It decides where judgment should be spent.

---

## 4. The evidence is not resolving into one clean story

> **Section brief.** Put conflicting public evidence in conversation. Resist the urge to choose either the productivity narrative or the sceptical narrative as settled fact.

One of the easiest mistakes available to anyone writing about AI at the moment is to gather only the evidence that lets them feel prescient. I could do that in either direction.

There is now enough public evidence to write a cheerful paper about the transformation. Anthropic’s internal study reports engineers using Claude heavily, self-reporting substantial productivity gains, working across a wider range of tasks, and attempting work that otherwise would not have been attempted. It also reports that debugging and code understanding are common uses, more common, in its sample, than new-feature implementation. [Anthropic, “How AI is transforming work at Anthropic”](https://www.anthropic.com/research/how-ai-is-transforming-work-at-anthropic)

There is also enough public evidence to write a satisfying corrective. METR’s randomised study of sixteen experienced open-source maintainers working on mature projects they knew well found that access to early-2025 AI tools made tasks take 19% longer, despite the participants’ expectation that the tools would make them faster. [METR, “Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity”](https://metr.org/Early_2025_AI_Experienced_OS_Devs_Study-paper.pdf)

Neither finding is disposable. Neither proves the whole case.

Anthropic is studying its own unusually capable people using unusually capable tools in an organisation that has every reason to learn good delegation. Much of its productivity evidence is self-report, and it says so. METR is unusually rigorous about a difficult real-world question, but studies a narrow population, familiar mature codebases, tasks of a particular size, and tools that were already moving quickly beneath the research. Its authors do not claim that agents are useless in other settings.

The tension is not an inconvenience in the evidence. It is the evidence.

What seems to be emerging is a less flattering, more useful proposition: the value of an agent is conditional. It depends on the task selected, the maturity of the model, the amount of tacit context, the quality of the environment, the strength of available checks, the reversibility of the change, and the team’s own learned skill at delegation. “AI productivity” is a single, tidy phrase that hides all of those variables and then asks us to make an organisational decision as though it had said something precise.

DORA’s current framing is helpful here. Its 2025 work describes AI as an amplifier of existing team and system conditions, and stresses the role of internal platforms, clear workflows, safety nets, and user focus. [DORA, “State of AI-Assisted Software Development”](https://cloud.google.com/blog/products/ai-machine-learning/announcing-the-2025-dora-report) Its earlier work also found a cautionary pattern: individual flow and productivity could improve while delivery throughput and stability moved in the wrong direction. [DORA, “Impact of Generative AI in Software Development”](https://dora.dev/research/ai/gen-ai-report/dora-impact-of-generative-ai-in-software-development.pdf)

That does not settle causation. It does name the thing I would now watch: not whether a person felt faster at their desk, but whether the whole system became better at producing outcomes without quietly transferring cost to QA, operations, customers, or the future team.

---

## 5. Comprehension is changing shape

> **Section brief.** Qualify the original paper’s concern about delegated comprehension. Agents can be instruments of understanding or substitutes for it; the difference is what remains with the team after the interaction.

I do not want to lose the original paper’s anxiety about comprehension. It came from a real intuition: writing, debugging, and changing a system are among the ways a person comes to understand it. If we casually delegate those encounters, we may wake up in a codebase that can be changed only by asking the same kind of system that produced it to change it again.

But the public accounts make a distinction I did not have language for when I began: there is **assisted understanding**, and there is **borrowed understanding**.

Assisted understanding is when an agent helps a person or team find their way into a system, formulate a better question, trace a dependency, compare alternatives, or make an otherwise hidden constraint visible. The agent is a tool of investigation. The team comes away more able to explain, repair, and extend the system than it was before.

Borrowed understanding is when an agent supplies a plausible explanation or a working change and nothing durable remains except confidence that it sounded right at the time. The team cannot say why the boundary exists, whether the test proved the important thing, what assumptions the implementation made, or where to look when reality differs from the answer. The agent may have been helpful in the moment. The system is less owned afterwards.

Anthropic’s study contains both futures in miniature. It reports people using agents frequently for debugging and code understanding, while also recording concern that some engineers are getting less practice and that colleagues may turn to the agent before turning to one another. [Anthropic, “How AI is transforming work at Anthropic”](https://www.anthropic.com/research/how-ai-is-transforming-work-at-anthropic)

That is why Knowledge Debt still belongs at the centre of the argument. I would now define it more carefully: not the mere presence of agent-written code, but the gap between what the system requires of its maintainers and the shared ability of those maintainers to recover the reasons, constraints, and consequences of changing it.

The countermeasure is not to prohibit assistance, and not to require performative line-by-line penance for every generated change. It is to leave behind things that survive the interaction: useful explanations, recorded decisions, tests that mean something, visible operational behaviour, shared review where the blast radius demands it, and deliberate opportunities for people to learn the judgment that agents cannot simply hand them.

---

## 6. What I would now ask of an organisation adopting agents

The original paper ended constructively because criticism without a practice is mostly a way of making oneself feel clearer than everyone else. I still believe the alternative is available. I can now say a little more concretely what it asks for.

It asks an organisation to begin with an outcome and an evidence plan, not merely an implementation prompt. What will change for the person who needs this? What must not break? What would persuade us that the result is good enough?

It asks for context that is discoverable rather than encyclopaedic: a small map, maintained sources of truth, decisions close enough to the code and the work that both the next engineer and the agent can find them. The document must remain a hypothesis, but a hypothesis with a home.

It asks people to use agents before commitment as well as after it: to investigate the system, generate competing approaches, identify missing questions, create small experiments, and make uncertainty visible. The agent is at its most valuable when it makes it cheaper to learn that the first confident answer was wrong.

It asks teams to constrain through invariants where they can. Not because every important rule can be reduced to a test, but because any important rule that *can* be made executable should not have to be rediscovered by a tired reviewer every third pull request.

It asks for autonomy scaled to verifiability and reversibility. Give agents room where the result can be checked cheaply and rolled back safely. Slow down where the evidence is weak, where the consequence is high, or where a local change quietly becomes an architectural decision.

And it asks that failure change the harness. A missed constraint should become a better check, a better map, a clearer boundary, a more honest decision record, or a smaller permission. Otherwise the team is not learning; it is simply accumulating anecdotes about why the agent cannot be trusted.

This is not a call for a new bureaucracy. It could become one, easily. A harness can ossify into the same elaborate control system this whole series is trying to resist. The live test is the old one: does it make revision easier when reality teaches us something, or does it make correction harder because the process has become more important than the thing being learned?

---

## Open register: questions I am keeping open

**“Better models will move the boundary.”**

Almost certainly. Nothing in this document should be read as a claim that the current division of labour is permanent. Models are improving at long-horizon work, context management, tool use, and verification. The relevant discipline is not to defend today’s boundary forever; it is to ask what evidence justifies moving it tomorrow.

**“Agent-to-agent review may be enough in some cases.”**

It may be. “A human in the loop” can become its own empty ceremony, no better than a sign-off gate. If a change is checked independently against strong evaluation suites in a reversible environment, human line review may add cost without reducing meaningful risk. The claim here is not that a person must read everything. It is that someone remains responsible for deciding when the available evidence is enough.

**“The harness is simply Big Design Up Front by another name.”**

It can be, if it is frozen, overgrown, and treated as complete. That is the danger. The difference is not the number of documents or checks. It is whether the system can learn: whether evidence from work alters the knowledge, constraints, and next decision rather than arriving too late to disturb the plan.

---

## Sources and notes

These are deliberately close to the claims above. This is a small starting register, not an argument by credential. Company accounts are reported as company accounts; the METR study is narrow but rigorous; DORA’s organisational findings are associative rather than causal. The point is to leave a trail a reader can inspect.

- OpenAI, [“Harness engineering: leveraging Codex in an agent-first world”](https://openai.com/index/harness-engineering/) (2026).
- OpenAI, [“How OpenAI uses Codex”](https://openai.com/business/guides-and-resources/how-openai-uses-codex/) (accessed 2026).
- METR, [“Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity”](https://metr.org/Early_2025_AI_Experienced_OS_Devs_Study-paper.pdf) (2025).
- Anthropic, [“How AI is transforming work at Anthropic”](https://www.anthropic.com/research/how-ai-is-transforming-work-at-anthropic) (2 December 2025).
- DORA, [“State of AI-Assisted Software Development”](https://cloud.google.com/blog/products/ai-machine-learning/announcing-the-2025-dora-report) (23 September 2025).
- DORA, [“Impact of Generative AI in Software Development”](https://dora.dev/research/ai/gen-ai-report/dora-impact-of-generative-ai-in-software-development.pdf) (April 2025).

---

## Closing note

The pendulum has not stopped. There are still organisations using agents to renew the old promise: complete knowledge first, execution second, certainty by the date on the slide. There are also organisations beginning to discover something more interesting: that cheaper production can shorten the distance between an idea and evidence, if they build the conditions that let evidence travel back.

That is the choice I can see more clearly now than when I started the first paper. The question is no longer only whether the agent can build from the prompt. Increasingly, it can.

The question is whether the surrounding system can teach it, constrain it, test it, observe it, correct it, and preserve what was learned when the work is over.

That surrounding system is where the next swing will be decided.
