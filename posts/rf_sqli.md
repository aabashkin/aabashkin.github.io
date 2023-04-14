# Security Vulnerabilities In Robotic Process Automation (RPA) / Low Code Technology: SQL Injection & The Return of Bobby Tables

![Comic](./img/xkcd.png)

<br>

- [Security Vulnerabilities In Robotic Process Automation (RPA) / Low Code Technology: SQL Injection \& The Return of Bobby Tables](#security-vulnerabilities-in-robotic-process-automation-rpa--low-code-technology-sql-injection--the-return-of-bobby-tables)
  - [Audience](#audience)
  - [Summary](#summary)
  - [Background](#background)
  - [Security Risks](#security-risks)
  - [A Modern Example with a Low Code / RPA Platform](#a-modern-example-with-a-low-code--rpa-platform)
  - [Prevention](#prevention)
    - [Parameterized Queries](#parameterized-queries)
    - [Stored Procedures](#stored-procedures)
    - [Escaping](#escaping)
    - [Input Validation](#input-validation)
  - [Future Challenges](#future-challenges)
  - [Additional Issues](#additional-issues)
  - [Further Improvements](#further-improvements)
    - [Building Additional Security Controls](#building-additional-security-controls)
    - [Raising Awareness](#raising-awareness)
  - [Conclusions](#conclusions)
  - [References](#references)
  - [Follow Me](#follow-me)
  - [Join the Conversation](#join-the-conversation)

<br>

## Audience
* Users of Robotic Process Automation (RPA) and Low Code (LC) technologies
* Product managers and engineers responsible for building these technologies
* Application / software security professionals

<br>

## Summary
* Robotic Process Automation (RPA) and Low Code (LC) technologies are exploding in growth due to their ease of use and wide variety of practical application for ordinary business users.
* This growth will create a massive number of new business technologists, aka citizen developers, within practically ever organization.
* These new technologies create a lot of exciting possibilities, but these possibilities carry some significant security risks as well.
* This blog post will examine a common type of vulnerability, SQL injection (SQLi), in the context of LC / RPA in order to demonstrate the issue at hand by using a concrete example.
* Guidelines around preventative measures and remediation will be presented.
* Future challenges and additional issues will be reviewed.
* Code and instructions for running examples can be found in the [GitHub repo](https://github.com/aabashkin/low-code-rpa-robot-framework-security-sql-injection-demo)

<br>

## Background

Low Code (LC) technologies and Robotic Process Automation (RPA) are exploding in growth due to their ease of use and wide variety of practical application for ordinary business users. What was once the domain of traditional software engineers is now open to the average "citizen developer".

Here are some [eye opening statistics](https://quandarycg.com/low-code-statistics/):
* "By 2025, organizations will build **70% of their new applications** using low-code or no-code platforms." [Source](https://www.gartner.com/en/newsroom/press-releases/2021-11-10-gartner-says-cloud-will-be-the-centerpiece-of-new-digital-experiences)
* "**26% of executives** believe low-code platforms are the most critical investment in automation." [Source](https://www.cio.com/article/193919/the-road-to-modern-delivery-low-code-development-market-speed-and-the-future-of-it.html#_ftnref2)
* "Gartner states that by 2023, there will be **4x more citizen developers than professional developers** at large enterprises." [Source](https://www.gartner.com/en/documents/4005412)

For more background on this growing technology trend, check out the following:
* [What Is Low Code? (Brief introduction video)](https://www.youtube.com/watch?v=4zrGil0fWng)
* [Why You Should Look Into Low Code (Deep dive)](https://www.youtube.com/watch?v=MBKeKso6mtw)
	
<br>
	
## Security Risks

The opportunities that come with these new technologies creates a lot of exciting possibilities, but these possibilities carry some significant security risks as well. The comic above was published many years ago and has become a cult classic among application / software security experts and security conscious software engineers. But for those readers who don't come from this background, let's take a moment to briefly review what's going on. For those who are already familiar with SQLi, feel free to skip this section.

The comic references a very common software vulnerability known as SQL injection (SQLi). As far as vulnerabilities go, this a particularly nasty one, given the various types of impacts it can have on a system. This vulnerability allows an attacker to potentially:
* Destroy data (the case in our comic)
* Insert new data
* Alter existing data
* Exfiltrate data
* Perform a denial of service (DOS) attack, rendering the system unusable
* Run system commands directly on the host of the database, creating a foothold for further attacks on other assets of the organization

In short, SQL injection can wreck havoc on any system. This has been known for a long time, and is the reason why so much ink has already been spilled on the topic. In fact, there is an [entire website](https://bobby-tables.com/) dedicated to this sole issue! In addition to this, the [OWASP Top Ten](https://owasp.org/Top10/), a project dedicated to enumerating some of most common and high impact security vulnerabilities in software, has included this issue in every single version since the origin of the project in 2003.

The good news is that SQL injection has become less prevalent over time, mostly due to improved software frameworks, security testing tools, and more awareness among software engineers. But while traditional software has been steadily improving, the rise of Low Code / RPA creates the potential for this vulnerability (and many others like it) to raise its ugly head once again.

In order to understand why this is so, we need to begin by reviewing how a typical SQL injection vulnerability occurs. Databases are often accessed by special commands known as SQL queries. For our example, let's say a school has a database where it stores some information about its students. If we want to insert a new student record into the database, we could create a query like this:

```sql
INSERT INTO Students (name) VALUES ('Alice');
```

The example above specifies the type of operation we want to perform (`INSERT`), the table in the database we want to add it to (`students`), as well as the field in the table (`name`), and finally the data itself (`Alice`).

Now let's make this example a bit more realistic. Typically, we wouldn't be writing code that has queries with a literal value such as `Alice`. Instead, we would create a dynamic query that could accept multiple values. We would gather student names from some kind of data source, such as a spreadsheet or a form on a website, and then pass that value into the SQL query before firing it off. The query would look something like this:

```sql
INSERT INTO Students (name) VALUES ('${student_name}');
```

In this example, the variable `${student_name}` represents the data that we got from our web form. So what's the issue here? Simply put, an attacker could submit some malicious input to our website form and completely alter the entire query. Going back to the comic, our cheeky parent decided to name their child `Robert'); DROP TABLE Students;--`. When that input is combined with our query, the final result looks like

```sql
INSERT INTO Students (name) VALUES ('Robert'); DROP TABLE Students;--');
```

The special characters `');` at the end of Robert create the effect of completing the first query, but then allowing *another* query to be added to the end. This query (`DROP TABLE Students;`) has the effect that you think it does.

This is just one example, but it should be apparent how dangerous it can be to combine untrusted user input with regular SQL queries. Our attacker now has the ability to do anything with the database that the application itself can do. This is the main reason why this type of vulnerability has been given so much attention over the past two decades.

<br>

## A Modern Example with a Low Code / RPA Platform

We've discussed traditional applications, but what about our new technologies? Let's have a look at how this issue can manifest itself using a popular open source Low Code / RPA platform called the
[Robot Framework](https://robotframework.org/) along with the [RPA Framework](https://rpaframework.org/) designed for it. Unlike a typical programming language, Robot Framework "has an easy syntax, utilizing human-readable keywords. Its capabilities can be extended by libraries implemented with Python, Java or many other programming languages. Robot Framework has a rich ecosystem around it, consisting of libraries and tools that are developed as separate projects."

Let's recreate the example from above in RF with the help of the [RPA.Database library](https://robocorp.com/docs/libraries/rpa-framework/rpa-database):

```RobotFramework
*** Tasks ***

Insert Students Into Database Using Dynamic Query
    @{list_students}=    Get Students From Excel File    resources/new_students.xlsx
    FOR    ${student_name}    IN    @{list_students}
        ${query}=    Format String    INSERT INTO students (name) VALUES ('{}');    ${student_name}
        Query    ${query}
    END

*** Keywords ***

Get Students From Excel File
    # Implementation omitted for brevity
    ...
    RETURN    ${list_students}
```
	
In this example, our data source is an Excel file called `new_students.xlsx`. We read this file using a custom function (known as a keyword) that we define called `Get Students From Excel File`. For the sake of simplicity, I've omitted the implementation of this keyword, all that you need to know is that it returns a list of student names from the Excel file name that we provide. To see the full implementation, please go to the [code repo](https://github.com/aabashkin/low-code-rpa-robot-framework-security-sql-injection-demo)

Next, we take that list and loop through it, generating a new SQL query for each student and sending the query to our database. This is where we encounter the SQL injection vulnerability from our initial example.

This line of code:
```RobotFramework
${query}=    Format String    INSERT INTO students (name) VALUES ('{}');    ${student_name}
```

Combined with Bobby's full name: 
```sql
Robert'); DROP TABLE Students;--
```

Becomes this query:
```sql
INSERT INTO Students (name) VALUES ('Robert'); DROP TABLE Students;--')
```


**To see this example in action, please refer to the instructions in the [code repo](https://github.com/aabashkin/low-code-rpa-robot-framework-security-sql-injection-demo).**

<br>

## Prevention

There are various ways to prevent this vulnerability.

These include:
* Parameterized queries
* Stored Procedures
* Escaping
* Input Validation
	
None of these techniques are novel or unique to our Robot Framework example. They are tried and tested methods of avoiding SQL injection. The only difference is how they are applied specifically within Robot Framework.

Let's go step by step.

<br>

### Parameterized Queries

When generating the query, replace the `Format String` keyword with the `Set Variable` keyword and replace the `VALUES ('{}')` placeholder with a `VALUES (\%s)` placeholder.

Next, add a `data` parameter containing untrusted input, `${student_name}` in this case, to the `Query` [keyword](https://robocorp.com/docs/libraries/rpa-framework/rpa-database/keywords#query).

The data parameter accepts [tuples](https://www.w3schools.com/python/python_tuples.asp), so even if you have a single input item you must still add an extra empty item at the end in order to execute the query. Simply providing `("${student_name}")` won't work, `("${student_name}", )` will.

```RobotFramework
*** Tasks ***

Insert Students Into Database Using Parameterized Query
    @{list_students}=    Get Students From Excel File    resources/new_students.xlsx
    FOR    ${student_name}    IN    @{list_students}
        ${query}=    Set Variable    INSERT INTO students (name) VALUES (\%s);  
        Query    ${query}  data=("${student_name}", )
    END
```

**Please note that every database has a different style of placeholder syntax.** Our example contains the syntax used by PostgreSQL. Other databases rely on a different syntax, so please refer to this [guide](https://bobby-tables.com/python) when necessary.

When we run this example, the platform recognizes the **clear separation between data and code**, thus eliminating the attacker's ability to manipulate the function of the query itself. This separation is the **core principle** for avoiding any type of injection vulnerability.

<br>

### Stored Procedures

Start by creating a stored procedure in the database.

```sql
CREATE PROCEDURE insert_student(student_name VARCHAR(255))
LANGUAGE SQL
AS $$
    INSERT INTO students (name) VALUES (student_name);
$$;
```

Next, utilize the `Call Stored Procedure` [keyword](https://robocorp.com/docs/libraries/rpa-framework/rpa-database/keywords#call-stored-procedure) in your code.

```RobotFramework
*** Tasks ***

Insert Students Into Database Using Stored Procedure
    @{list_students}=    Get Students From Excel File    resources/new_students.xlsx
    FOR    ${student_name}    IN    @{list_students}
        @{params}     Create List   ${student_name}
        Call Stored Procedure   insert_student  ${params}
    END
```

**Please note that due to an [issue](https://www.psycopg.org/docs/cursor.html?highlight=callproc#cursor.callproc) in the `psycopg` Python module this approach is currently incompatible with any Postgres 11+ database.** Instead, one can use the `Query` keyword along with the `CALL` instruction. However, in order to properly generate this query string one must use parameterized queries as described in the previous section. For the sake of simplicity it may make more sense to just use parameterized queries and avoid the extra overhead of obtaining access to the database and creating a stored procedure. However, if a stored procedure already exists in your use case and you would like to leverage it, see the example below.

```RobotFramework
*** Tasks ***

Insert Students Into Database Using Stored Procedure (Postgres)
    @{list_students}=    Get Students From Excel File    resources/new_students.xlsx
    FOR    ${student_name}    IN    @{list_students}
        ${query}=    Set Variable    CALL insert_student (\%s)
        Query    ${query}  data=("${student_name}", )
    END
```

This example also conforms to the principle of separating code and data. Manipulation of the query is no longer possible.

<br>

### Escaping

Escaping neutralizes any special control characters in our input, thus preventing it from altering the query. Apply escaping by surrounding the `{}` placeholder with double dollar signs `$$`.

Since Robot Framework recognizes a dollar sign followed by a curly brace as a variable declaration we have to add an extra backslash `\` to `$${}$$`, so the final result should be `$\${}$$`.

```RobotFramework
*** Tasks ***

Insert Students Into Database Using Query With Escaping
    @{list_students}=    Get Students From Excel File    resources/new_students.xlsx
    FOR    ${student_name}    IN    @{list_students}
        ${query}=    Format String    INSERT INTO students (name) VALUES ($\${}$$);    ${student_name}
        Query    ${query}
    END
```

**Please note that every database has a different style of escaping.** Our example contains the syntax used by [PostgreSQL](https://www.postgresql.org/docs/15/sql-syntax-lexical.html#id-1.5.3.5.9.7.2). Please refer to your database's documentation for guidance relevant to your use case.

Just like the techniques mentioned above, this example separates code and data and secures our query.

<br>

### Input Validation

As an example, let's say that we've decided that valid student names for our database should contain only letters from the Latin alphabet and hyphens. We use this rule to create a regular expression `^[a-zA-Z-]+$` and apply it to each student name using the `Should Match Regexp` keyword.

```RobotFramework
*** Tasks ***

Insert Students Into Database Query With Input Validation
    @{list_students}=    Get Students From Excel File    resources/new_students.xlsx

    ${pattern}=    Set Variable    ^[a-zA-Z-]+$

    FOR    ${student_name}    IN    @{list_students}
        TRY
            ${result}=    Should Match Regexp    ${student_name}    ${pattern}
            ${query}=    Format String    INSERT INTO students (name) VALUES ('{}');    ${student_name}
            Query    ${query}
        EXCEPT   * does not match *    type=glob
            Log To Console    Input validation failed. Student name does not meet requirements.\n
        END
    END
```

**Please note that input validation is most effective when heavily constrained.** The example above could actually become vulnerable if we allowed additional special characters, such as `'`, `)`, and `;`. Deciding if input validation is sufficiently constrained is a difficult topic, even for tech savy users. For this reason, it is recommended to avoid relying entirely on input validation, and instead use it as a secondary security control. That being said, input validation provides benefits beyond just security, such as ensuring data accuracy and consistency, so it is still a good idea to consider it.

<br>

## Future Challenges

As you can see, low code technologies are susceptible to some of the same risks as traditional applications. However, there are additional challenges in this space:

* **Awareness** - Training software engineers to avoid these types of issues is already a challenge. In person training requires significant resources and runs into scaling issues. Delivering self paced training is more affordable, but may not produce as much engagement. If the number of citizen developers keeps growing at a rapid pace then so will the scale of the problem.

* **Mindset** - On average, it is easier to motivate software engineers to care about security, especially if you frame the problem as a code quality issue. Producing well written software is the main job responsibility of an engineer, and people like to take pride in their work. Presenting engineers with an opportunity to improve their code can be an effective way to grab their attention. On the other hand, citizen developers are more likely to be focused solely on solving business problems and are less likely to be focused on the technical aspects of the solution. These types of users understandably have a "just get it done" mentality.

* **Tooling** - Mature solutions for scanning traditional programming languages and frameworks (known as static analysis) exist in large numbers. Unfortunately, far fewer solutions are currently available for low code projects. Even if software engineers introduce a security vulnerability into their code, there is a significant amount of opportunities for automated tools to catch the mistake. This is not quite the case for citizen developers. Another related issue is scale. Just like the issues with scaling awareness training, there are considerable issues with scaling scanning tools for low code projects because of their sheer number and non-comformance to traditional software development lifecycles. Even if we had the scanning tools available, integrating them into the development lifecycle of citizen developers would be its own challenge.

<br>

## Additional Issues

Databases are not the only attack surface. Additional vulnerabilities may exist in automation which interacts with:
* HTTP requests (Server Side Request Forgery)
* Command lines (Remote Code Execution)
* Files (Local File Inclusion)
* HTML generation (Cross Site Scripting)
* Archives (Zip Bombs)
* LDAP (LDAP injection)
* SMTP (SMTP injection)
* and so on...

Bottom line, many of the issues that affect typical applications can affect Low Code / RPA as well. Stay tuned for more blog posts about these topics in the future.

<br>

## Further Improvements

### Building Additional Security Controls
When I first began my security research in this area, the [RPA.Database library](https://robocorp.com/docs/libraries/rpa-framework/rpa-database) did not support parameterized queries. I quickly identified this as an area for improvement and submitted a [pull request](https://github.com/robocorp/rpaframework/pull/899). Thankfully, the PR was quickly reviewed, merged, and [released](https://updates.robocorp.com/release/BSwnn-rpaframework-2240).

Looking at the list of additional issues above, I believe that over time it is very likely that we will discover more improvements that can be made. Just as in other areas of application security, a collaborative effort to solving this problem is key.

The folks at [Robocorp](https://robocorp.com/), the company which maintains [RPA Framework](https://rpaframework.org/), have been very receptive to my work and provided all the support that I needed in order to be successful. If you are considering contributing your own work, please don't hesitate. The [community slack](https://robocorp-developers.slack.com/) is a good place to start for validating ideas and figuring out next steps. Hopefully, other low code platforms follow this example.

### Raising Awareness
Back in the early 2000's, application security was still an obscure principle. Many people simply didn't realize what type of risks were associated with insecure code. In order to raise awareness about the issue, the first version of the [OWASP Top 10](https://owasp.org/www-project-top-ten/) was released in 2003. Since then, the project has undergone many revisions and today is considered one of the most authoritative sources of information about application security risks.

Using this as a model for success, the [OWASP Low-Code / No-Code Top 10 Project](https://owasp.org/www-project-top-10-low-code-no-code-security-risks/) was born. Just like the original Top 10, it will undoubtedly evolve over time, and hopefully become a useful resource for all Low Code users.

I call on all interested parties to contribute to this effort in whatever ways they can. A great place to start is the [Slack channel](https://owasp.slack.com/archives/C02C6RU6G10) and [Google Group](https://groups.google.com/g/owasp-no-code-low-code). A special thanks goes out to all of the folks that have contributed so far!

<br>

## Conclusions 

In short, Low Code is here, and it's here to stay. When it comes to adopting Low Code in your organization it is simply a question of when, not if.

Monitoring the security space in Low Code is crucial, and will provide may exciting opportunities for future security research and innovation.

<br>

## References
[SQLi Robot Framework example source code on GitHub](https://github.com/aabashkin/low-code-rpa-robot-framework-security-sql-injection-demo)

[Robot Framework / RPA Framework Security Cheatsheet](https://github.com/aabashkin/cheatsheets/blob/main/low-code-rpa-robot-framework-security-sql-injection.md)

[OWASP Low-Code / No-Code Top 10 Project](https://owasp.org/www-project-top-10-low-code-no-code-security-risks/)

<br>

## Follow Me

[Twitter](https://twitter.com/AntonAbashkin)

[LinkedIn](https://www.linkedin.com/in/antonabashkin/)

<br>

## Join the Conversation

[OWASP Low-Code / No-Code Slack](https://owasp.slack.com/archives/C02C6RU6G10)

[OWASP Low-Code / No-Code Google Group](https://groups.google.com/g/owasp-no-code-low-code)
