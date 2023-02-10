---
title: Chapter 6. Managing Multiple Teams
link: the-managers-path-chapter-6
hidden: true
---

<div class="lang-switch" style="text-align: right">
    <a href="#">EN</a>
    <a href="/the-managers-path-chapter-6-zh/">中文</a>
</div>

<div style="font-family: 'Zilla Slab', serif;">

Welcome to the world of multiple-team management! We’re going to talk about managing multiple teams before we talk about managing managers, because while those things are related, they don’t necessarily coincide. You probably have tech leads reporting to you now, though, and juggling the work of directly managing more than three or four people with the process of understanding details about what’s happening across a couple of teams probably means one important thing: you’re not writing (much, any, production) code.

When I created the career ladder for my previous job, the director of engineering role was usually the place where a person would start to manage multiple large teams. Let’s review some of the description from my engineering ladder:

<div style="margin-left: 1em; margin-bottom: 1em;"> 

*The engineering director is responsible for a significant area of the technology team. The engineering director typically leads engineers across multiple product areas, or multiple technology functions. Both tech leads and individual contributors report into them.*

*The engineering director is not generally expected to write code on a day-to-day basis. However, the engineering director is responsible for their organization’s overall technical competence, guiding and growing that competence in the whole team as necessary via training and hiring. They should have a strong technical background and spend some of their time researching new technologies and staying abreast of trends in the tech industry. They will be expected to help debug and triage critical systems, and should understand the systems they oversee well enough to perform code reviews and help research problems as needed. They should contribute to architecture and design efforts primarily by serving as the technically savvy voice that asks business and product questions of the engineers on their teams, ensuring that the code we are writing matches the product and business needs and can scale appropriately as those needs grow.*

*The engineering director is primarily concerned with ensuring smooth execution of complex deliverables. To that end, they focus on ensuring that we continually evaluate and refine our development/infrastructure standards and processes to create technology that will deliver sustained value to the business. They are responsible for creating high-performance, high-velocity organizations, measuring and iterating on processes as we grow and evolve as a business. They are the leaders for recruiting, headcount management and planning, career growth and training for the organization. As necessary, directors will manage vendor relationships and participate in the budgeting process.*

*The impact of an engineering director should reach across multiple areas of the technology organization. They are responsible for creating and growing the next generation of leadership and management talent in the organization, and helping that talent learn how to balance technical and people leadership and management. They are obsessed with creating high-functioning, engaged, and motivated organizations, and they are expected to own retention goals within their organization. Additionally, engineering directors are responsible for strategically balancing immediate and long-term product-/business-focused work with technical debt and strategic technical development.*

*Directors are strong leaders who set the example for cross-functional collaboration both between technology and other areas of the company, and across divisions of technology. The goal of this collaboration is to create both a strategic and tactical tech roadmap that tackles business needs, efficiencies and revenue, and fundamental technology innovation. The director is a very strong communicator who can both simplify technical concepts to explain them to nontechnical partners and explain business direction to the technology team in a way that inspires and guides them. Directors of engineering help to create a positive public presence for Rent the Runway tech and are capable of selling the company and their area to potential candidates.*

*Due to their breadth of exposure to both technology and the business drivers, directors are responsible for guiding the goal-setting process for all of the teams in their organization, helping these teams articulate goals that support both business initiatives and technology and organizational quality.*

</div>

I took pains to make sure that we called out the fact that engineering directors would not necessarily be writing code every day, because I believe that it is very difficult for a person responsible for hands-on management of multiple teams to write code. Your schedule, by this point, has probably moved away from “maker” and firmly into “manager.” Between your 1-1s, meetings with other engineering leads, team planning sessions, and sessions with your peers in product management or other business functions, you’re probably quite busy. Be realistic about your schedule at this point. If you don’t have solid blocks of time to dedicate to it and you can’t realistically guarantee solid blocks of time at least a few days a week, any code you write is going to be very slow-going.

Fortunately for us, there are ways to stay hands-on that don’t require writing a lot of production code. Code reviews are a good thing to stay in practice with, at least as a secondary reviewer. If you created systems when you were more hands-on, stay engaged with those systems, because you’ll remember the details better than most, and you can help engineers working in those systems with code reviews and questions. Debugging and production support are also valuable. How you stay hands-on depends on your skill set. If you were not a strong debugger before you went into management, jumping into incidents may be more annoying than useful. You may be more helpful doing pair programming, or fixing minor bugs or features. So often we diminish these small efforts as not worthwhile, but they’re very good at keeping you in tune with the feeling of software development and showing your teams that you are willing and able to help out with the day-to-day in a valuable way.

The risk of going hands-off is greatly amplified if you don’t spend enough time coding before moving into this role to get yourself deeply, fluently comfortable with at least one programming language. I advocate strongly that you spend the time to gain mastery of programming before moving into management. For me, this took about 10 years, including my undergraduate and graduate degrees. You may do it faster than I did, but scrutinize yourself carefully in this regard. Do you feel fluent enough in at least one programming language to productively contribute to a good code base written in it, after a limited time spent getting up to speed in the basics of writing it, using a standard development environment, and working in standard frameworks and libraries? Eventually even the deepest knowledge will atrophy, but fluency in working in a language (which includes comfort with its standard tools, libraries, and runtimes) is something that sticks with you for a long time.

Useful fluency also requires an ingrained understanding of what it means to work productively in such a language, hopefully on a team with other people building production software. Without this sense of the rhythms of building software, you’ll struggle with one of the critical parts of the job at this level: debugging team issues and keeping your teams producing quality software smoothly.

Finally, even if you don’t intend to write much code, I strongly advise you to keep at least a solid half-day once a week completely free from meetings or other obligations, and try to use this time at least partially on some creative pursuit. You might write blog posts for your engineering blog, prepare conference talks, or participate in an open source project. Do something to scratch that creative itch, which can otherwise be hard for you to scratch as a manager.

> <div style="text-align: center;font-size: 1.2em;font-weight: bold;margin-bottom: 8px;">ASK THE CTO: I MISS CODE!</div>
> 
> *I’m managing two complex teams and my management responsibilities are forcing me to step back from technical responsibilities. I find that I miss code terribly. Is this a sign that I shouldn’t be a manager?*
> 
> Almost everyone who goes from a heavily hands-on technical role into management has a transition period where they question frequently whether they’ve made a mistake. Furthermore, many worry that they’re losing all of their valuable skills in the process. Ask yourself whether you’ve internalized the idea that management is not a job. The tech industry is filled with people who despise management, thinking it’s not as important a job as writing code. But management is a job, it is a necessary and important job, and in particular, it’s your job right now.
> 
> Writing code is full of quick wins, especially for the experienced developer. You make tests pass, you see new features come to life, you get something to compile, you fix a problem. Management has fewer obvious quick wins, especially for new managers. It’s natural to feel some longing for simpler times, when it was just you and your computer and you didn’t have to deal with all these messy, complicated humans. You probably felt some similar nostalgia for your school days when you first started working full-time, because by the time you left school you knew exactly what to expect from it. It’s OK to feel nostalgia for the simpler times, and a little bit of fear for what you’re giving up. But you can’t do everything all at once. Becoming a great manager requires you to focus on the skills of management, and that requires giving up some of your technical focus. It’s a tradeoff, and one you’ll have to decide if you’re up to making.

## Managing Your Time: What’s Important, Anyway?

When you have so many management duties that you have little time to code, you can start to feel like your day has been taken hostage by the whims of others. You start to see the meetings pile up: the one-on-ones, the planning meetings, the status updates. Standups. Sit-downs. Fight fight fight!

Wait, no—no fighting in the war room!

It’s time—now—for you to figure out how to manage your time. Otherwise, you’ll find yourself with days gone by and little to show for them. You still have responsibilities as a manager. You still have deliverables that require you to do more than sit in a meeting—things like setting goals for the team, helping your product team put details on the product roadmaps, and making sure that an assigned task actually got finished. That last one, following up on task completion, can become one of the biggest time sinks and distractions in your day if you aren’t careful.

Time management is a personal thing. Some people are very organized, and those people develop complex strategies for managing their calendars and to-do lists. I applaud these people. I am not generally one of them. However, I have found the ideas in David Allen’s book *Getting Things Done* [^first] to be useful to think about, and I recommend reading it even if you don’t adopt the whole process.

In the meantime, my general time-management philosophy will serve you well no matter what your tactics might be. Managing your time comes down to one important thing: understanding the difference between *importance* and *urgency*. Pretty much all of your tasks will fall on a graph of these two elements. Roughly, they are in one of four quadrants (see Table 6-1).

|  | **Not urgent** | **Urgent** |
| - | - | - |
| **Important** | Strategic: make time | Obvious work |
| **Unimportant** | Obvious avoid | Tempting distractions |

<div style="text-align:center; font-style: italic">Table 6-1. Prioritizing your time</div>

If it’s important and urgent, you’re doing it. You know what I’m talking about. There’s a major outage that you’re helping to fix. Performance review write-ups are due tomorrow. You want to make an offer to a great candidate who has another competing offer expiring in two days. If you drop the ball on a task in this category, you lose something tangible. It’s unlikely that you’re blind to the need to get these sorts of things done.

A big part of the challenge of time management emerges when you start to lose the sense of importance. Urgency is often more clearly felt than importance. Responding to email is a good example. It’s easy to get sucked into email as a distraction, because the red dot tells you there’s something new, and it feels urgent to you to acknowledge it. And yet how often is email really urgent? Email is probably the worst vehicle for conveying urgent, time-sensitive information. It feels urgent, but it isn’t urgent. This is why so many precise time-management tips encourage reading and responding to email at specific times of the day. We also tend to substitute *obvious* for *urgent* in determining something’s value. If a meeting is on your calendar, it’s obvious where you should be at that time, but is that meeting really urgent, or are you using it to avoid thinking about the best way to use your time?

There are many things that feel urgent that aren’t. The whole of the internet, for example. News, Facebook, Twitter. Chat can feel urgent, but chat is almost as bad as email for communicating truly urgent and important information in the case of a collocated team. We’ve moved a lot of communication in the modern tech workplace out of email into chat systems like Slack and HipChat. This has benefits and drawbacks, but it’s important to note that moving communication is not the same thing as eliminating communication. The words and information keep flowing, they just move to different places, and you can get even more distracted by the ever-moving trickle of information in chat.

It’s likely that you’re spending a lot of your time on things that are urgent but only slightly important, and sacrificing things that are important but not urgent. One example of an important but not urgent task is actually preparing for meetings so that you can guide them in a healthy way. Healthy meetings require involvement from all parties, and a culture that favors short but productive meetings requires that participants do some up-front work to come to the meeting prepared. As a manager of multiple teams, you can win back a lot of time by pushing an efficient meetings culture down to your teams. Hold people accountable to prepare in whatever way makes sense. Ask for agenda items up front. Any sort of standard meeting that involves a group of people, whether it’s planning, retrospective, or postmortem, should have a clear procedure and expected outcomes.

One of the major changes at this level compared to the previous level is that your boss will expect you to be mature enough to manage yourself and your teams independently. This means that your manager trusts you to proactively deal with all those important but not urgent things before they become urgent, and especially before they become urgent for your manager. No one will tell you how to manage your calendar to give yourself the time to do this. I’ve seen managers fail at this point because they just could not juggle all the different tasks in an organized fashion.

Meetings can fall into that urgent but not important category, and you may decide to simply not attend them where you’re not clearly needed. Be very careful with over-deploying this strategy at this particular level of management. The responsibility of keeping your teams successfully moving forward and happily engaged is on your shoulders. When you stop going to all of their internal meetings, you run the risk of missing out on the very clues that will help you catch problems early—a major one being the existence of too many boring meetings. During meetings, look around the room at your team and notice their engagement. If half of them are falling asleep, staring off into space, on their phones or laptops ignoring the proceedings, or otherwise disengaged, the meeting is wasting their time. Your attendance at these meetings is partially to pay attention to the dynamics and morale of your team. A happy team will feel energized and engaged. An unhappy or unmotivated team will feel drained or bored.

Back to matters important but not urgent. Thinking about the future is high on this list. There are undoubtedly things that you know you should do but have put off. Perhaps it’s writing job descriptions for roles you’re hiring for. Perhaps it’s developing a hiring plan at all. It might be reviewing the current work on a project to make sure no obvious problems are creeping in, or talking to a manager on another team where there is conflict or a difference of opinion on how to move forward on a shared issue. It might be cultivating the list of things that are important but you haven’t thought about in a while, so you know what to focus on. If you don’t set aside some time to focus on these issues, they’ll sneak up on you in negative ways. As a manager of multiple teams, you’re responsible for balancing breadth and depth of thinking, for knowing the details of your teams today but also looking at where you need to be going in the future and what you need to get there.

As you navigate your new obligations, start to ask yourself: How important is the thing I’m doing? Does it seem important because it is urgent? How much time have I spent this week on urgent things? Have I managed to carve out enough time for things that are not urgent?

> <div style="text-align: center; font-size: 1.2em; font-weight: bold; margin-bottom: 8px;">THE HARDEST, SHORTEST LESSON OF BECOMING A MANAGER</div>
> 
> As a manager, I have this mental list of things about what my team needs. Things that I’m monitoring, things that I’m trying to fix, things that I’m trying to find for them. It’s my job to understand what is going on and what the team as a whole needs to be effective.
> 
> Maybe you can look at the state of things and say, “We have a deadline right now, and what we need is another engineer for the next month. That engineer is me.”
> 
> But more likely you look at the state of things and realize that what your team needs is a manager. Because you need to hire X more people. Because Y has a lot of potential but needs some coaching. Because product or design or some other team hasn’t given you what you need, so you need to go and get it. Because process is important, and the process you have is insufficient or just plain wrong.
> 
> If your team needs a manager more than they need an engineer, you have to accept that being that manager means that you by definition can’t be that engineer. I know some people manage both, but you need to decide, if you’re going to suck at one, which one that will be.
> 
> I feel bad when I suck at being an engineer, but sucking at being a manager would be a choice I inflicted on other people. That’s not fair.
> 
> So at the end of another day when I feel like I didn’t write enough code and I have no way to quantify what I’ve achieved, I tell myself I was being as good a manager as I know how to be. And that has to be enough for today.
> 
> Cate Huston

## Decisions and Delegation

How do you feel at the end of the day these days? If you’re like many new full-time managers, you probably feel quite drained. Even though you didn’t write a lot of code—or any code!—all day, when you get home you find yourself with no energy to decide what to eat for dinner, no energy for hobbies, and the desire to eat comfort food, drink a beer perhaps, and stare blankly at the computer or TV until it’s time to go to bed.

The first several months of managing multiple teams can feel like a death march, even when your hours are not excessive. Your once-focused attention gets sliced and diced between the various meetings that pepper your day. I lost my voice repeatedly during my first few months managing multiple teams; I was totally unused to talking so much every single day. A friend of mine recently became a director of engineering, and she had to start having an assistant order her lunch because she discovered that she would forget to eat—and had no energy to decide what to eat when she realized she needed food.

So, first, the bad news: the only way out of this situation is to go through it. In fact, I would expect most people to go through this for a while. If you haven’t experienced it at all, either count yourself extremely lucky, or double-check to make sure you’re really paying attention to everything that needs your attention. In my experience both going through this transition and managing people in it, if you don’t feel a little bit overwhelmed, you’re likely missing something.

The best way to describe the feeling of management from here on out is plate spinning. If you’re not familiar with it, plate spinning is a fancy form of juggling where the juggler has several poles, each with a plate spinning on top of it. The juggler must attend to each plate before it slows down enough to fall off the pole. Your plates are the people and projects you’re overseeing, and your job is to figure out how much attention each one needs at what time. It’s important that you approach this spinning with a student’s mind. You’re still learning how to spin plates, and you’re going to drop some on the floor because you’ve neglected them for too long. Honing your instincts about when to touch which plate is the name of the game.

Now for the good news: you’ll get better at this over time. Your instincts will improve. You’ll start to recognize the early warning signs of projects that are going poorly, people who are getting ready to quit, and teams that are underperforming. I recommended in the last section that you think carefully about dropping out of meetings, and part of the reason is that those meetings are where you learn what healthy and unhealthy dynamics look like. This is also why I strongly advise you maintain your practice of regular, reliable 1-1 meetings with everyone who reports directly to you. If you have too many people, you may need to shorten those meetings or hold them biweekly instead of weekly, but skipping 1-1s because you’re too busy with other things is a great way to miss the warning signs of an employee who is going to quit.

I called this section “Decisions and Delegation”—so where does delegation fit in? Delegation is the primary way you claw yourself out of the feeling of having too many plates spinning at once. As tasks come at you, ask yourself: do I need to be the person who completes this work? The answer may depend on a few factors (see Table 6-2).

| |Frequent|Infrequent|
|-|-|-|
|**Simple** |Delegate |Do it yourself|
|**Complex**|Delegate (carefully)|Delegate for training purposes|

<div style="text-align:center; font-style: italic">Table 6-2. Deciding when to delegate or do it yourself</div>

The degree of complexity and the frequency of the task can act as guides to determining whether and how you should delegate.

### Delegate Simple and Frequent Tasks

If the task is simple and frequent, find someone to whom you can hand it off. Examples of simple and frequent tasks might include running daily standups, writing up a summary of the teams’ progress each week, or conducting minor code reviews. Your tech leads or other senior engineers can take responsibility for these tasks and probably won’t even need training to do so.

### Handle Simple and Infrequent Tasks Yourself

If it’s faster to do something yourself than it would be to explain it to someone else and it rarely needs to be done, roll up your sleeves and do it, even if you deem the task beneath you. This can be anything from booking the occasional conference ticket for your team to running the script that generates quarterly reports.

### Use Complex and Infrequent Tasks as Training Opportunities for Rising Leaders

Tasks like writing performance reviews and making hiring plans are yours alone. However, these are also the skills you will want to pass on to rising managers. You may have a tech lead sit with you to write the performance review for an intern, or have a senior engineer provide feedback on how many new people he believes would be needed to support a project next year. Ask for help from above on these tasks until you feel comfortable doing them, but once you feel comfortable, start pulling in rising leaders to learn how they are done.

### Delegate Complex and Frequent Tasks to Develop Your Team

Tasks like project planning, systems design, or being the key person during an outage are the biggest opportunity you have to grow talent on your team while also making the team run better. Strong managers spend a lot of their time developing members of their teams in these areas. Your goal is to make your teams capable of operating at a high level without much input from you, and that means they’ll need individuals who can take over these complex tasks and run them without you around.

Are your teams learning how to operate independently, or are you keeping them dependent on you for critical functions? List the tasks that you and only you really know how to do for the team. Some of them may be appropriate, like writing performance reviews or making hiring plans, but many of them are important to teach your teams to accomplish themselves. Project management. Onboarding new team members. Working with the product team to break down product roadmap goals into technical deliverables. Production support. These are all skills members of your team need to learn. Teaching them may take time up front, but in the long run it will save you time. Not only that, but teaching your team how to do these things is part of your job. It is your responsibility, as a manager, to build up the talent in your organization, and to help your people learn new skills they’ll need for the next stages of their careers.

Delegation is a process that starts slow but turns into the essential element for career growth. If you teams can’t operate well without you around, you’ll find it hard to be promoted. Develop your talent and push decisions down to that talent so that you can find new and interesting plates to learn how to spin.

> <div style="text-align: center; font-size: 1.2em; font-weight: bold; margin-bottom: 8px;">ASK THE CTO: WARNING SIGNS</div>
> 
> *I’ve experienced a couple of times now when teams have struggled unexpectedly, and a person has quit without warning. Are there any warning signs I can look out for to catch these issues earlier?*
> 
> There are definitely signals that you start to notice after you’ve been managing for a while. Here are some I’ve learned to spot:
> 
> - **The person who is usually chatty, happy, and engaged suddenly starts leaving early, coming in late, taking breaks to leave during the workday, staying quiet in meetings, and not hanging out on chat.** This person is either having a major personal issue or getting ready to quit. Usually, people will tell someone when there is a personal issue (such as a sick relative, relationship problems, or health issues), but not always. If this happened right after a major adjustment, such as a promotion, a team reorganization, or other event, the person may feel that she was overlooked. Regardless of the reason, you may want to have an honest conversation and try to get to the root of the issue before she resigns.
> - **The tech lead claims that everything is going well, but skips your 1-1s frequently and rarely provides detail in his status updates.** This person may be hiding something. Often what he’s hiding is that the progress is going far slower than he anticipated, or he’s building something outside of the scope of the project. Help him create a clear project plan early and set expectations for how to adjust that plan when things change, so that it’s harder for him to hide a lack of progress. Also help him clarify the project’s goals and scope, which can be daunting for some new tech leads. You may have experienced something similar when managing new hires who were in over their heads. This is also related to the person who spends a lot of time advocating for new languages/platforms/processes instead of finishing her work.
> - **The team has absolutely no energy at all in their meetings.** In fact, the meetings feel like a total slog, with the product manager and tech lead doing all of the talking while the rest of the team sits silently or speaks only when called upon. A lack of engagement in meetings tends to mean the team isn’t engaged by the work or do not feel like they have a say in the decision-making process.
> - **The team’s project list seems to change every week, depending on the customers’ whims that day.** This team hasn’t thought about its goals beyond pleasing customers, and may need better product or business direction.
> - **A small team internally seems very fragmented in understanding; the engineers profess ignorance about systems they don’t work on and lack the curiosity or openness to learn about those systems.** This team is more strongly identified by their day-to-day work and the systems they touch than by the larger team or the company. They may be resistant to changing their systems based on the needs of the larger team or the business.

## Challenging Situations: Strategies for Saying No

A manager’s job involves making it easy for her employees to get things done by creating fertile environments in which work can happen. She focuses her team so that they can do what they do best. She cultivates camaraderie and friendship on the team, and helps people learn new skills. In all these things, she is an enabler, a coach, and a champion.

But to create this environment, she sometimes must say no. She must say no to the team. She must say no to her peers. She must even say no to her boss. Each of these nos is hard, in its own way, and a strong manager must develop effective strategies for saying no. Here are a few that I have identified.

### “Yes, and”

Saying no to your boss rarely looks like a simple “no” when you’re a manager. Instead, it looks like the “yes, and” technique of improvisational comedy. “Yes, we can do that project, and all we will need to do is delay the start of this other project that is currently on the roadmap.” Responding with positivity while still articulating the boundaries of reality will get you into the major leagues of senior leadership. This type of positive no is a pretty hard skill for most engineers to master. We’re used to articulating the downsides of projects, and getting out of the knee-jerk “no, that can’t happen” habit is hard. Start mastering the “yes, and” strategy for saying no, particularly when interacting with your boss and peers, and see how it often transforms contentious disagreements into realistic negotiations for priority.

### Create Policies

When it comes to your team, you want to help them understand what it takes to get to “yes.” Perhaps you are dealing with an engineer who wants to switch to a new programming language for a project, one that your team doesn’t use. He has some great arguments as to why this language is the perfect tool for the job, but you’re reluctant to add a new tool just because it’s perfect. You might be tempted to just say no, give the reasoning, and leave it, and sometimes that will work. But you may find yourself saying the same “no” over and over again, giving the same reasons. “No, we need to have more people who know that language; we need to understand what it means to put that language into production.” “No, we need to have standards for logging; we need to think about what testing would look like.” When you start repeating yourself, you have the basis for a reasonable policy. That policy consists of the hard requirements that must be met in order to say yes, and some guidelines for thinking about the decision. Making a policy helps your team know in advance the cost of getting to “yes.”

### “Help Me Say Yes”

Policies are useful, but they don’t cover every case. The next strategy, “help me say yes,” is similar to policy writing, but works better for the one-off instances where there is no clear policy. Sometimes you’ll hear ideas that seem very ill-considered. “Help me say yes” means you ask questions and dig in on the elements that seem so questionable to you. Often, this line of questioning helps people come to the realization themselves that their plan isn’t a good idea, but sometimes they’ll surprise you with their line of thinking. Either way, curious interrogation of ideas can help you say no and teach at the same time.

### Appeal to Budget

When it comes to both your team and your peers, one tactic that you can use is appealing to time and budget. Lay out the current workload in plain terms, and show how there is little room to maneuver. Sometimes this is coupled with “not right now,” another somewhat passive-aggressive way of saying no. “Not right now” implies that you might agree with the idea but can’t do it at this moment, so maybe you’ll get to it in the future. This is frequently true, and so it’s easy to fall back into “not right now” mode. But as I discussed earlier, when you give an implied promise that “not right now” means that you’ll seriously do something “later,” you need to be sure that later can actually happen.

### Work as a Team

Speaking of your peers, there will be times when you and your peers (particularly across functions—i.e., your product or business peers) will need to act together to say no. This can apply to a no at any level. Sometimes you will use your technical authority to say no in a way that is beneficial to the product team. Sometimes you will appeal to the finance department to help you say no to certain budget excesses. Playing good cop/bad cop can be a little bit dishonest, so use this sparingly, but it can be useful to lend your authority to a no and then be able to call in the favor when you need support for your own no in the future.

### Don’t Prevaricate

When you know that you need to say no, it’s better to say it quickly than to delay and drag out the process. If you have the authority to say no, and you don’t believe something should happen, do yourself a favor and don’t agonize over the process. You’ll be wrong sometimes, so when you discover that you were too quick to say no, apologize for making that mistake. You won’t have the luxury to carefully investigate and analyze every decision, so practice getting comfortable with the quick no (and the quick yes!) for low-risk, low-impact decisions.

> <div style="text-align: center; font-size: 1.2em; font-weight: bold; margin-bottom: 8px;">ASK THE CTO: MY TECH LEAD ISN’T MANAGING</div>
> 
> *I have a tech lead who was supposed to be overseeing one of our junior engineers on a project to rewrite our app from Objective-C to Swift. I just found out that the junior engineer still hasn’t created a project plan, and hasn’t answered any of the feedback I gave in the design review. How do I get the tech lead to manage this without me having to step in?*
> 
> Delegation failures happen. It sounds like your tech lead doesn’t understand that you’re holding her responsible for making sure that the junior engineer follows up on design feedback and creates a project plan. So the first step is to ask the tech lead why these things haven’t happened yet.
> 
> The answer you will get is likely to be a combination of things. One, the tech lead is busy with her own work and didn’t remember to follow up with the junior engineer. This happens, and you’ll have to remind her that mentoring and overseeing this person’s work needs to be scheduled in with her code and other responsibilities.
> 
> Two, the tech lead may not know how to push the junior engineer when he doesn’t want to commit to a schedule. Ask her how she’s tried to get the information from him, and see whether you have an opportunity to suggest different approaches. Sometimes new tech leads are reluctant to push people for project plans because they don’t feel that they have authority and are flustered when they ask for something and the other person just never delivers.
> 
> The best thing to do here is to work with your tech lead to give her the skills and confidence to ask for reports from other members of the team. It will be slower than stepping in and asking for them yourself, but you’ll teach the team to respect her requests and teach her how to lead the team independently.

## Technical Elements Beyond Code

Management at this level gets confusing. We hire managers based partially on their technical skills, but many of us think of this job as not really “technical.” After all, the manager is probably not writing much code or doing much systems design, right?

Assuming that the job at this level becomes essentially nontechnical is a mistake. It turns out there’s more to running effective engineering teams than pure management skills, and management at this level will require you to learn some new skills that are easiest to learn if you understand the practice and discipline of software engineering. You’re going to turn your technical focus now to observing and improving the systems of work that your developers are operating within. In particular, you now need to develop an eye for the technical health signals for the overall team. But what are those health signals?

The popular management book *First, Break All the Rules* [^second] discusses several questions you can answer to help predict team productivity and satisfaction. Among them are:

- Do I know what is expected of me at work?
- Do I have the materials and equipment I need to do my work right?
- Do I have the opportunity to do what I do best every day?

For most engineers, the answer to these questions can be discerned by the speed and frequency with which they push code. If the work they need to do is clear, they know what code to write. If the tools, tickets, automation, and process are available and easy to use, they are able to get the code written. And if they’re not distracted by excessive meetings or drowning in support and incident management, they’ll get a chance to write code every day. These health signals—frequency of code releases, frequency of code check-ins, and infrequency of incidents—are the key indicators of a team that knows what to do, has the tools to do it, and has the time to do it every day.

## Measuring the Health of Your Development Team
When you’re focusing on the health of your development teams, put on your technical hat to design systems and processes that will keep things moving. Create the tools that developers need to do their jobs. Help them focus so they can easily figure out what needs to happen next. Interrogate every process to determine the value it should provide, and always ask yourself if it can be automated further. Consider these ways to measure the health of your team.

### Frequency of Releases

In [Chapter 5](/the-managers-path-chapter-5) I talked about how a common dysfunction of technical teams is not shipping code, and release frequency is the most direct measure of this. If your company doesn’t appreciate the value of releasing code frequently, I’m sorry. In this modern era, frequency of code change is one of the leading indicators of a healthy engineering team. Great engineering managers of product-focused teams know how to create environments where their teams can move fast, and part of moving fast requires breaking work down into small chunks. Even if your company doesn’t appreciate it, you must work to help your team achieve the best release frequency possible for their product. Lest you think this doesn’t apply to you because you’re building a product (say, a database) that can’t release frequently, I’m sure there is a complete artifact pushed to a beta/developer testing environment that provides a similarly valuable measure in terms of frequency and stability.

Why don’t you release more frequently? Take a look at your team. If they don’t release continuously, or daily, what does the release process look like? How long does it take? How often has something gone wrong in the past few months regarding releases? What does it look like when things go wrong? How often have you had to delay or roll back a release due to problems? What were the impacts of that delay or rollback? How do you determine if code is ready to go into production? How long does that take? Who is primarily responsible for determining this?

I bet if you honestly take a look at a team that isn’t releasing frequently, you’ll see cracks. The process of performing a release takes a long time. Engineers don’t feel ownership over their code quality, and they leave all of that work to a QA team, which creates a lot of back-and-forth communication delays. Rolling back code in the case of a bad release takes a long time. Things go wrong in the process of releasing that lead to incidents in production (or broken development builds). A whole host of ills in a team come from not being able to release frequently.

Now, you might say, “Thanks for the advice, but I don’t have the time to work on this with the product roadmap we have to deliver.” Or, “Our systems aren’t designed to be released frequently.” Or, “It’s just not that important that we change things that frequently.”

Here’s the thing. Is your team working to its full capacity? Are your engineers challenged and growing? Is your product team excited by the progress you’re making? Are people able to spend most of their time on writing new code and evolving the systems? If so, great. Ignore me. You’ve got it under control. If not, you have a problem, and you’re ignoring that problem at your peril.

It’s important to remember that, as a technical leader, while you may not be writing code much, you’re still responsible for the technical side of getting work done. You’re also responsible for keeping your team happy and productive, and often the solution to this is not cheerleading or paying them better or praising them more, but instead enabling them to be more productive, challenging them to go faster and do better work, and helping them find the time they need to make their work more interesting. You have to be the advocate and push for technical process improvements that can lead to increased engineer productivity, even if you’re not implementing them all yourself.

The beauty of pushing for more frequent releases is that it often uncovers a host of interesting challenges. There’s no one true way to increase release frequency because frequency problems will be somewhat unique from team to team. You’ll almost certainly need to solve some automation elements. Developer tooling to enable feature toggles that make sense for your code bases is another frequent challenge. Thinking about architecting code to move forward without breaking backward compatibility, rolling upgrades of systems, and implementing small changes instead of giant patches—all of these things may need addressing. You’re responsible for leading the effort here, even if you don’t do the work. Push for time away from the product roadmap to support increasing engineering productivity, and set goals for the team that inspire them to move faster.

### Frequency of Code Check-ins

It’s hard to have an agile team that doesn’t understand the value of breaking its work down into small chunks. You’ll likely need to teach this skill to new hires out of college, but even senior developers sometimes need a push here. I’m not going to advocate for any particular software development methodology, but I find that engineers who don’t write tests often have a harder time breaking down their work, and learning how to do test-driven development (even if they don’t actually practice that on a daily basis) can help them get better at this skill.

I focus on this topic because it can be very uncomfortable for you as a new manager to tell people who may have been writing code for as long or longer than you have that their style needs updating. Conflict avoidance runs deep in most of us, and matters of what feels like personal style can be particularly difficult. If your company expects fast-moving product development, engineers who want to go off for weeks and write code alone without pushing it into shared version control will slow your team down and cause problems. You’re not managing a research team. (Are you? Skip this section, then!) It’s OK for you to have the expectation that work in progress is regularly updated.

### Frequency of Incidents

How stable is the software being produced by the team? Is the quality improving, getting worse, or staying the same? Determining the level of software quality you need for the product you’re building and adjusting that measure over time is a technical challenge for you, the manager, to help address. If you’re building a brand new product for a small but growing business, it may be more important to focus on features over stability. On the other hand, if you own mission-critical systems, stability and incident minimization may be your top priority. The goal here is to balance risk in such a way that neither incident frequency nor incident prevention turns into a job that takes developers away from writing code for days at a time.

You may work for a company that has developers support the code or systems they write. This process has some downsides; significantly, expecting members of a team to frequently be on-call nights and weekends is a huge contributor to burnout. Despite that risk, it has the upside of putting the best people to help fix a problem in the role of responding to it. As a manager, you may be tempted now to take yourself out of this role. I sympathize, but if your team is set up to do its own incident management, you should be moving yourself into the role of escalation support. You won’t necessarily manage incidents as frequently, but you’ll be expected to be available more often in case the person supporting the systems needs you.

Analysis around incident management should include the question, “Is our current setup enabling my team to do what they do best every day?” Incident management, when it becomes merely reacting to incidents rather than working to reduce them, can turn into a task that diminishes your team’s ability to do what they do best. Engineers go on-call, they get burned out and exhausted from handling the deluge of problems and getting nothing done but fixing the consequences of incidents, and then they hand off the job to the next poor sap on the rotation. If that describes your team’s approach to incident management and on-call, your team is not able to do what they do best every day, and every time they go on-call they probably hate their jobs a little bit more. In this case, as a leader you probably want to focus on providing time to actually design systems that are more stable, or writing code to fix the recurring incidents as they arise.

An overemphasis on incident prevention can also reduce your team’s ability to do what they do best every day. Overfocusing on building systems that are defect-free, or pushing for error prevention by slowing down the development process, is often almost as bad as moving too fast and releasing unstable code. When risk reduction turns into weeks of manual QA, excessive and slow code reviews, infrequent releases, or a drawn-out planning process, the increased analysis can leave developers idle and restless, without necessarily reducing the risk of incidents.

## Good Manager, Bad Manager: Us Versus Them, Team Player

*Diana has just joined a midsize startup to run the long-neglected mobile team. She was told coming in that the team was a mess, so her first step is to quickly hire in a bunch of new people who worked for her at BigCo. They aren’t quite a culture fit, and the team quickly turns into a clique of developers who see themselves as better than the rest of the organization. While the technology has improved, they seem to be clashing a lot with the product team, and ultimately the apps aren’t evolving very quickly. After a year, Diana gets fed up with the company and quits. The rest of her new team soon follows, leaving the company right back where it started.*

It can be hard for new managers to create a shared team identity. Many of them default to an identity built around the specifics of their job function or technology. They unite the team by emphasizing how this identity is special as compared to other teams. When they go too far, this identity is used to make the team feel superior to the rest of the company, and the team is more interested in its superiority than the company’s goals. Rallying a team in this way is a   that is vulnerable to many dysfunctions:

- **Fragile to the loss of the leader.** In-group teams tend to be very fragile to the loss of their leader. When you hire a manager who builds a clique, that clique is likely to dissolve and leave the company if the manager leaves the company. This problem makes it that much harder to address the problems that the manager is causing by creating a clique in the first place.
- **Resistant to outside ideas.** In-groups tend to be resistant to ideas that do not come from those within the group. This means that they miss opportunities to learn and grow. The lack of growth for members of the team often causes the best members of the team to leave not just the group, but the company. Because they believe they’re in the best group but they still find themselves bored, they don’t appreciate the growth they could find just by switching to a new team.
- **Empire building.** Leaders who favor an us-versus-them style tend to be empire builders, seeking out opportunities to grow their teams and their mandates without concern for what is best for the overall organization. This often results in competition with other leaders for headcount and control of projects.
- **Inflexibility.** These groups tend to struggle against change that comes from outside the group. Reorganizations, cancelled projects, and shifting focus all can cause breaks in the core parts of their identity. Whether it’s a move from functional groups to cross-functional teams, delaying the iPad application, or prioritizing a new product, the change can devastate the fragile bonds of the team to the company.

As a manager, be careful about focusing on your teams to the exclusion of the wider group. Even when you have been hired to fix a team, remember that the company has gotten this far because of some fundamental strengths. Before you try to change everything to fit your vision, take the time to understand the company’s strengths and culture, and think about how you’re going to create a team that works well with this culture, not against it. The trick is not to focus on what’s broken, but to identify existing strengths and cultivate them.

*Neil has also joined a startup where things are chaotic. While he can see that he needs to change the team, he moves cautiously to fire people and takes the time to make sure that new hires are always vetted by someone who’s been with the company for a while. He spends a lot of time working closely with his peers in product, and proposes a path forward that emphasizes cross-functional collaboration. He focuses on setting clear goals and communicating them to his team. Things start out slow, but over time the entire organization feels stronger and both the technology and the product have improved dramatically.*

Durable teams are built on a shared purpose that comes from the company itself, and they align themselves with the company’s values (see “[Applying Core Values](/the-managers-path-chapter-9/#ApplyingCoreValues)” in [Chapter 9](/the-managers-path-chapter-9/) for more on this topic). They have a clear understanding of the company’s mission, and they see how their team fits into this mission. They can see that the mission requires many different types of teams, but all of the teams share a set of values. By creating a strong and enduring alignment between the team, its individuals, and the overall company, this *purpose-based binding* makes teams:

- **Resilient to loss of individuals.** While the clique is fragile, especially to the loss of the leader, the purpose-driven team tends to be very resilient to the loss of individuals and leadership. Because they’re loyal to the mission of the larger organization, they can see a path forward even through loss.
- **Driven to find better ways to achieve their purpose.** Purpose-driven teams are more open to new ideas and value changes that can help them serve their purpose better. They care less about the source of an idea than its merit in achieving their goals. The members of these teams are interested in learning from others outside their function, and they actively seek out chances to collaborate more broadly to create the best results.
- **First-team focused.** Leaders who are strong team players understand that the people who report to them are not their first team. Instead, their first team is their peers across the company. This first-team focus helps them make decisions that consider the needs of the company as a whole before focusing on the needs of their team.
- **Open to changes that serve their purpose.** The collaborative leader understands that changes will happen to serve the wider purpose. Teams will change structure and people will need to move to where the business needs are. With that knowledge, these leaders create teams that are more flexible and understanding of frequent change in service of the larger vision.
  
Getting clarity about the purpose of your team and your company can take time. In startups especially, there is often some confusion about the current goals and even sometimes the underlying mission. In the case where the goals are fuzzy and the mission is unclear, do your best to understand the company culture and think about how you can set your teams up to work well within that culture. By collaborating across teams and across business functions, your teams will come to understand the bigger picture and appreciate their mission as part of that picture.

## The Virtues of Laziness and Impatience

I love Larry Wall’s idea that “laziness, impatience, and hubris” are virtues of engineers, as he articulated in *[Programming Perl](http://shop.oreilly.com/product/9780596004927.do)*.[^third] These virtues persist into leadership, and learning how to channel these traits into advantages is something I encourage all managers to do.

As a manager, when you are dealing with people one-on-one you probably don’t want to be impatient, of course. Impatience can be rude if it’s directed at individuals. And you don’t want to seem lazy, because there’s nothing worse than working for a manager who seems to be taking it easy while you kill yourself to deliver projects. But impatience paired with laziness is wonderful when you direct it at processes and decisions. Impatience and laziness, applied to process, are the key elements to focus.

As you grow more into leadership positions, people will look to you for behavioral guidance. What you want to teach them is how to focus. To that end, there are two areas I encourage you to practice modeling, right now: figuring out what’s important, and going home.

I can’t stand watching people waste their energy approaching problems with brute force and spending time rather than thought. And yet, any culture where you are encouraged to work excessive hours all the time is almost certainly doing just that. What is the value in automation if you don’t use it to make your job easier? We engineers automate so that we can focus on the fun stuff—and the fun stuff is the work that uses most of your brain, and it’s not usually something you can do for hours and hours, day after day.

So be impatient to figure out the nut of what’s important. As a leader, any time you see something being done that feels inefficient, question it: Why does this feel inefficient to me? What is the value in the thing we are doing? Can we deliver that value faster? Can we strip down this project into something simpler and get it done more quickly?

The problem with this line of questioning is that often when managers ask whether something can be done faster, what they explicitly or implicitly want to know is whether the team can work harder or longer hours to deliver it in fewer days. This is why I encourage you to develop and show the value of laziness. Because “faster” is not about “the same number of hours but fewer total days.” “Faster” is about “the same value to the company in less total time.” If the team works 60 hours in a week to deliver something that otherwise would’ve taken a week and a half, they haven’t worked faster, they’ve just given the company more of their free time.

This is where going home comes in. Go home! And stop emailing people at all hours of the night and all hours of the weekend! Forcing yourself to disengage is essential for your mental health, believe me. Burnout is a real problem in the American workforce these days, and almost everyone I know who has worked sustained excess hours has experienced it to some degree. It’s terrible for individuals, terrible for their families, and terrible for teams. But this isn’t just about preventing your own burnout—it’s about preventing your team’s burnout. When you work later than everyone else, when you send those emails at all hours, even if you don’t expect your team to respond to those emails or work those hours, they see you doing it and think it’s important. And that overwork makes them less effective, especially at the detailed knowledge work that engineers need to perform.

When you’re a newish manager and you haven’t figured out the tricks to do your job effectively, you might find yourself needing to work more hours to get it all done. That’s OK for a little while. But I encourage you to figure out a way to work those hours without encouraging your team to do so, or making them feel obligated to be on your schedule. Queue up the weekend and overnight emails for the next workday. Put your chat status as “away” in off hours. Take a vacation, and don’t answer email during that time. And constantly ask yourself the same questions you ask your team: Can I do this faster? Do I need to be doing this at all? What value am I providing with this work?

Laziness and impatience. We focus so we can go home, and we encourage going home because it forces us to constantly focus. This is how great teams scale.

## Assessing Your Own Experience

- When was the last time you reviewed your schedule for things that you’re doing but that aren’t providing a lot of value for you or your team? Look back on the past couple of weeks. Look forward to the next couple of weeks. What did you accomplish, and what do you hope to accomplish?
- If you are still writing code, how does this fit in with the rest of your schedule? Are you doing it after hours? What’s driving you to continue to spend this time?
- What was the last task you delegated to a member of one of your teams? Was it simple or complex? How is the person you delegated to handling the new task?
- Who are the rising leaders of your teams? What is your plan for coaching them to take on bigger leadership roles? What tasks are you giving them to prepare them for more responsibility?
- Does the process of writing, releasing, and supporting code seem to function smoothly on your teams? When was the last time there was a noticeable incident with part of this process? What happened, and how did the team respond to it? How often does the process encounter these exceptional conditions?
- When was the last time you pushed your team to cut the scope of a project? When you cut scope do you cut features, technical quality, or both? How do you decide?
- When was the last time you sent email after 8 pm or on the weekend? Did the person you sent that email to respond? Did you need him or her to respond?

[^first]: 1. David Allen, *Getting Things Done: The Art of Stress-Free Productivity* (New York: Penguin, 2001).
[^second]: 2. Marcus Buckingham and Curt Coffman, *First, Break All the Rules: What the World’s Greatest Managers Do Differently* (New York: Simon & Schuster, 1999).
[^third]: 3. Tom Christiansen, brian d foy, Larry Wall, and Jon Orwant, *Programming Perl*, 4<sup>th</sup> edition (Sebastopol, CA: O’Reilly, 2012).

[Next Chapter: Managing Managers](/the-managers-path-chapter-7/)

</div>