# Skill 3 — Exam-Style Multiple-Choice Generation

## English

**Purpose**: practice under conditions that actually resemble the exam, rather than generic "test your knowledge" questions. DVA-C02 questions are almost entirely scenario-based ("a developer needs X, given constraints Y and Z, which approach...") with plausible-sounding wrong answers designed to catch common misconceptions — and that specific format needed to be practiced, not just the underlying content.

**Input**: a module's notes, plus the pattern of what real DVA-C02 questions look like (scenario framing, answer-choice style, the kind of "almost right but for one detail" distractor design that the real exam uses).

**Process**: generate multiple-choice questions modeled on that same scenario-plus-distractor structure in English — a short situational setup, four options where more than one initially sounds plausible, and the correct answer hinging on a specific detail from the notes (a number, a behavior, a "this service does X but not Y" distinction). Not simple recall-the-definition questions.

**After each answer**: the skill doesn't just mark right/wrong. It states the correct answer, explains why it's correct, explains why the plausible-sounding distractors are wrong (since those are the ones designed to catch misconceptions), and — specifically when the answer given was wrong or the reasoning behind a correct answer seemed shaky — names the underlying knowledge gap the miss points to, not just the single fact that was missed. This feeds directly into Skill 4, which looks for patterns across these named gaps.

**Output**: practice questions used per module and, later, mixed across modules to simulate the actual exam's topic spread, each paired with an answer explanation and (where relevant) a named gap.

**Design notes**: the value of this skill depended entirely on the distractors being genuinely plausible rather than obviously wrong. Matching real exam phrasing and difficulty was the explicit goal, not just generating any multiple-choice question about the topic. Naming the gap rather than just the correct answer was what made the feedback usable for review, instead of just a score.

---

## 中文

**用途**：在真正接近考试的条件下练习，而不是那种泛泛的"测测你懂不懂"式题目。DVA-C02 的题目几乎全是场景题（"某开发者需要 X，在 Y、Z 约束下，应该用哪种方案……"），错误选项设计得听起来很有道理，专门用来抓住常见的误解——这种特定的题目形式本身需要专门练习，不只是练底层知识。

**输入**：某个模块的笔记，加上真实 DVA-C02 题目长什么样的样式模式（场景化的题干写法、选项的风格、真实考试那种"看起来快对了，但差一个细节"式干扰项的设计方式）。

**处理过程**：按照同样的"场景+干扰项"结构生成英文选择题——一段简短的情景设定、四个选项里不止一个乍看起来都说得通，正确答案取决于笔记里的某个具体细节（一个数字、一种行为、一个"这个服务能做 X 但不能做 Y"的区分点）。不是简单的"背定义"题。

**答题之后**：这个技能不只是判断对错。它会给出正确答案，解释为什么这个答案是对的，解释那些听起来也有道理的干扰项为什么是错的（因为这些选项本来就是为了抓住常见误解设计的），并且——尤其是在答错了、或者虽然答对但推理过程显得不扎实的时候——指出这次失误反映出的具体知识点不足，而不只是说错在哪个事实点上。这一步直接为 Skill 4 提供输入，Skill 4 会在这些被指出的知识点缺口里找模式。

**输出**：按模块使用、后期跨模块混合的练习题，每道题都配有答案解析，相关时还会配上指出的知识点不足。

**设计说明**：这个技能的价值完全取决于干扰项是不是真的有迷惑性，而不是明显错误。让题目风格和难度尽量贴近真实考试，是这个技能明确追求的目标，而不是随便生成一道关于这个主题的选择题就算了。指出具体的知识点缺口，而不只是给出正确答案，才是让这个反馈能真正用于复习的关键，而不只是一个分数。


## Example Question
A developer is building a photo-sharing application that automatically enhances images uploaded by users to Amazon S3. When a user uploads an image, its S3 path is sent to an image-processing application hosted on AWS Lambda. The Lambda function applies the selected filter to the image and stores it back to S3.

If the upload is successful, the application will return a prompt telling the user that the request has been accepted. The entire processing typically takes an average of 5 minutes to complete, which causes the application to become unresponsive.

Which of the following is the MOST suitable and cost-effective option which will prevent the application from being unresponsive?

A. Configure the application to asynchronously process the requests and use the default invocation type of the Lambda function.

B. Use AWS Serverless Application Model (AWS SAM) to allow asynchronous requests to your Lambda function.

C. Configure the application to asynchronously process the requests and change the invocation type of the Lambda function to Event.

D. Use a combination of Lambda and Step Functions to orchestrate service components and asynchronously process the requests.
