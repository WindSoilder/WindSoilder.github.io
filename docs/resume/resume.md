Keith Chen
Back-End Engineer

yonghengzer@gmail.com
https://github.com/WindSoilder

# Profile
I am a experience software engineer, mainly working as a backend engineer, programming in python and rust.  With strong experience in back-end development, design RESTFUL api independently.  As well as collaborate with team member using agile methodologies.

Personally, I'm care about unit testing, documentation, writing readable and testable code.

Fan of solving problems, writing code, and contributing to open source projects.

# Skills
programming languages: python, rust.

Back-End Framework and Databases: flask/fastapi, celery, mongodb, mysql, redis, rabbitmq(mainly used with celery)

Testing and Automation Frameworks: pytest, unittest.mock

Solid basic computer knowledge includes Computer network(HTTP/TCP/IP), Data structure, basic algorithm.

Experience with kunernetes and docker basic usage.

# Works
1. Ricequant (Permanent)
Sofeware Engineer, Apr 2018 - Present

1. Redesign and write internal web service named performance attribution for customer, accomplished:
- required much less memory(from 32G to 8G)
- gained much more performance(reduce 70% of the runtime in the general scenario)
- get more extendable and easily testing service.

2. Prompt and rewrite a new C-S based mongodb synchronize, it supports sync internal mongodb data to customer.  User can specific which database, which collection to sync, after a careful design, the syncer accomplished:
- during client full synchronized progress, the origin database can provide services as usual(no directly increasing workload)
- customer get reliable and realtime update from original database.
- less mongodb instance is used in the company(from 4 to 1).

3. I'm also working on something includes:
- Finance service developments
- design RESTFUL api
- makes some profile and performance improvement for existing code base.

Tools & Technologies: Python3(pandas + pytest + FASTAPI + celery + mypy), Rust, RESTFUL api design, distributed computing, rust, mongodb, Jira, Jenkins, Confluence

2. Dell software (Permanent)
Associate Software Developer, Jun 2016 - Apr 2018

Firstly I'm using python + robotframework to build UAT automation testing for product, and involve in improving testing code performance, which reduces running time from 2hours to 1hour.

Then involve in product development, which mainly using C# and powershell.

Tools & Technologies:
Python, Robotframework, test automation, agile methodologies, C#, AWS, Confluence, Jira, Jenkins

3. Alibaba Group (Internship)
Test Development Engineer, Jun 2015 - Aug 2015

Mainly working on testing security product.

The test process includes: Functional Testing, Performance Testing, Stability Testing

Tools & Technologies:
Python, test automation

# Projects
- Nushell (Apr 2022 - present)
Nushell is an new type of shell, it mainly contains the following feature:
1. multi platform supports, Nushell works with LinuxOS, MacOS, Windows natively.
2. Like powershell, everything is structured data, so we don't need to treated everything as bare string.
3. Provide powerful plugin system, so it's easy to extend nu.
4. It also provides easy to use Nu language.

Currently I'm acting as a core team member, what I have done are:
1. Finishing missing unit tests, which makes user change their code in the future more confidently.
2. solve several bugs in nushell, mainly make nushell works better with external commands.
3. develop a plugin which makes nushell compatible to many binary format, which includes ttf, png, bmp and so on...
4. investigate and develop an nushell lib which empowers background job
5. add signature information when user want to get help on one command, it makes commands much easier to use.
6. review pull requests from other contributors
