# Behavioral Interview Preparation

> Sources: Personal experience and project history
> Updated: 2026-04-17

## Overview

Chuẩn bị câu trả lời cho các câu hỏi behavioral interview phổ biến, sử dụng phương pháp STAR (Situation, Task, Action, Result). Nội dung dựa trên kinh nghiệm thực tế 5+ năm làm fullstack/frontend developer.

---

## Giới thiệu bản thân

> **Tip**: Giữ ngắn gọn (60-90 giây). Nêu: tên, kinh nghiệm, project gần nhất, điểm mạnh, lý do quan tâm vị trí.

Hi, I'm Ngoc, 27 years old. I am currently working as a Full-stack Developer at Ewoosoft Viet.

Recently, I've led frontend development for a global **Group Purchase Campaign Platform**, using Next.js 16, Tailwind, and Shadcn. I focus on performance, scalability, and a consistent design system, while building a modular architecture with **monorepo**. I also collaborate closely with backend teams (using NestJS, PostgreSQL) to ensure smooth end-to-end integration.

Over the past **5+ years**, I have worked as a full-stack and front-end developer, including experience outsourcing for **FPT Software**, insourcing for **Viettel Security**, as well as role at **Ewoosoft Viet**.

I have hands-on experience with React.js, Next.js, Node.js/NestJS, MongoDB, PostgreSQL, Docker.

I also received the **"Best Employee of the Year 2025"** award for my contributions to architecture optimization and the integration and sharing of AI tools within the workplace.

Looking ahead, I am very excited about this Fullstack Developer position because it perfectly aligns with my expertise and career goals.

### Các điểm nhấn khi giới thiệu

| Điểm nhấn                | Chi tiết                                     | Tại sao quan trọng                  |
| ------------------------ | -------------------------------------------- | ----------------------------------- |
| Led frontend             | Group Purchase Campaign Platform, Next.js 16 | Chứng minh khả năng lead            |
| Monorepo architecture    | pnpm + Turborepo                             | Cho thấy hiểu scalable architecture |
| Cross-team collaboration | Backend NestJS/PostgreSQL                    | Fullstack mindset                   |
| Award                    | Best Employee 2025                           | Được công nhận                      |
| AI tools integration     | Sharing AI tools trong team                  | Proactive, cập nhật technology      |
|                          |                                              |                                     |

---

## Tại sao rời công ty cũ?

> **Tip**: Không nói xấu công ty cũ. Focus vào growth và new challenges.

I decided to leave my previous company to look for **new challenges and opportunities** where I can further grow my skills as a full-stack developer, especially in **building scalable systems** and working across both frontend and backend.

### Nếu bị hỏi thêm chi tiết

- "Tôi đã học được rất nhiều ở công ty cũ, nhưng cảm thấy đã đến lúc tìm kiếm môi trường mới để phát triển thêm."
- "Tôi muốn làm việc với các hệ thống lớn hơn, phức tạp hơn."
- Tránh: lương, conflict, nói xấu manager/đồng nghiệp.

---

## Thử thách và bài học (STAR)

> **Tip**: Đây là câu hỏi phổ biến nhất. Chuẩn bị 2-3 stories STAR. Story dưới đây về technical challenge.

### Story: Modernization của codebase legacy

**Situation**

In my previous project, a dental clinic management system, I was working on a long-maintained codebase built with **outdated technologies** like React 16 and Node.js 16. The project had accumulated a lot of **technical debt** over time, including **unused style submodules**, legacy code patterns, and a large number of unnecessary dependencies. This significantly slowed down development and made onboarding and maintenance difficult.

**Task**

My responsibility was to improve developer experience and system performance **without breaking existing functionality**, while also preparing the codebase for future scalability.

**Action**

Tôi đề xuất và dẫn dắt effort modernization:

1. **Build tooling**: Migrate frontend từ Create React App sang **Vite**, upgrade Node.js v16 → v20.
2. **Codebase cleanup**: Dùng tool **Knip** để audit → tìm và xóa hơn **1,000 unused files** và khoảng **50 unnecessary dependencies**. Dọn sạch legacy submodules và refactor outdated patterns.
3. **Frontend optimization**: Tối ưu data fetching bằng Apollo's **cache-and-network policy**, gom GraphQL queries, và implement **optimistic updates** để cải thiện UX.

**Result**

- Local dev startup: dropped from **1 phút 30 giây → 1.2 giây** (giảm ~98%).
- Production build: improved by **~5 lần**.
- Codebase cleaner, easy to maintain → **reducing onboarding time** for new developer.
- Frontend performance và UX cải thiện rõ rệt nhờ data handling hiệu quả hơn.

### Các con số cần nhớ

| Metric | Trước | Sau | Cải thiện |
|---|---|---|---|
| Dev startup time | ~90s | ~1.2s | 98% faster |
| Production build | baseline | 5x faster | — |
| Unused files removed | — | 1,000+ files | — |
| Unused dependencies | — | ~50 packages | — |

---

## Câu hỏi behavioral phổ biến khác

### What do you know about our company NAB?

I know NAB is Australia’s largest business bank. NAB Vietnam is leveraging partnerships with AWS and Microsoft. I'm excited to join a team that builds high-scale products for over **8 million customers** while maintaining a flexible, **Hybrid** work culture.

### Agile/Scrum

Yes, I have experience working with Agile/Scrum. I have participated in sprint planning, daily stand-ups, sprint reviews, and retrospectives. I’m comfortable working in cross-functional teams and collaborating closely with product managers, designers, and QA to deliver features iteratively.

### Tell me about a time you disagreed with a team member

> **Framework**: Acknowledge disagreement → explain your reasoning → show compromise/resolution → positive outcome.

*Gợi ý story*: Tranh luận về cách tiếp cận architecture (monolith vs modular) → present data/benchmarks → team đồng ý sau khi thấy kết quả → project thành công.

### Tell me about a time you failed

> **Framework**: Honest about failure → what you learned → how you applied it.

*Gợi ý story*: Estimate sai timeline, deploy gây lỗi production → learned to test more carefully, add CI checks → chưa bao giờ lặp lại.

### How do you handle tight deadlines?

> **Framework**: Prioritize → communicate → deliver MVP first.

*Gợi ý*: "I break the work into must-have vs nice-to-have. Communicate early if timeline is at risk. Deliver a working MVP first, then iterate."

### How do you keep up with new technologies?

*Answer*: "I maintain a personal knowledge wiki where I research, take notes and synthesize information from articles, books, and courses. I also actively integrate AI tools into my workflow and share learnings with my team — which contributed to my Best Employee award."

### What specifically interests you about this role at NAB, and what value do you believe you can bring in your first three to six months? How would you measure success?

*Answer*: "What excites me most about this Full-stack developer role at NAB is the opportunity work on large-scale, mission-critical digital banking platform that power everyday financial experiences for millions people. 
In the first 3 to 6 months I can bring immediate value by:
- Quickly delivering production-ready features using the modern stack I've been shipping daily (like Nextjs, Reactjs, Tailwind/Shadcn, backend pattern with Nestjs/Nodejs)
- Helping scale component libraries and architecture
- Driving performance and developer experience improvements - for example I migrated CRA project to Vite (cutting startup time from 90 seconds to 1-2 seconds and build time by 5x)"

---

## Tips chung cho behavioral interview

1. **STAR method**: Mọi câu trả lời đều có Situation, Task, Action, Result.
2. **Quantify results**: Dùng con số cụ thể (98% faster, 1000+ files, 5x improvement).
3. **"I" not "We"**: Nói rõ **bạn** làm gì, không nói chung chung "team chúng tôi".
4. **Practice out loud**: Đọc to câu trả lời, mỗi câu 1-2 phút, không quá dài.
5. **Prepare 3 stories**: Mỗi story có thể dùng lại cho nhiều câu hỏi khác nhau.
6. **Be honest**: Nếu không biết → nói "I haven't experienced that, but here's how I would approach it."

## See Also

- [AWS Services Interview Q&A](../aws/aws-services-interview.md)
- [React Fundamentals Interview](../react/interview/react-fundamentals-interview.md)
- [React Hooks Interview](../react/interview/react-hooks-interview.md)
- [React Components Interview](../react/interview/react-components-interview.md)
- [React Architecture Interview](../react/interview/react-architecture-interview.md)
