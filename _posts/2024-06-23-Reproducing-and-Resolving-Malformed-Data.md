---
layout: post
title: Reproducing and Resolving Malformed Data
cover-img:
  - https://img.freepik.com/free-vector/abstract-banner-with-modern-pixel-design_1048-15764.jpg?t=st=1719187797~exp=1719191397~hmac=f4953dc7b47c6b5c20b73a7f4175a7abaa83c3727c3775cffce3689751f5fe06&w=2000: "Image by kjpargeter. Source: https://tinyurl.com/kjp-photo License: Free"
tags:
  - star-story
---
As a software engineer, I frequently work to help popular websites more effectively serve their users. Data is a big part of that, since insight into user behavior is needed to improve performance and content. One day, my manager alerted me to a problem with our site data collection. While running a spot check, he had observed data with incorrect formatting. I was tasked with [identifying the root cause](https://www.elastic.co/what-is/root-cause-analysis) of the bad data, and writing a ticket to resolve the problem.

The first action I took was to quantify the prevalence of the problem. I wrote a simple #SQL script to understand how frequently this formatting error occurred. Year-to-date, about 11% of all samples had the incorrect formatting problem. I forwarded the measurement to my manager and we agreed that this merited action.

The first challenge lay in reproducing the issue. I read the JavaScript code generating the data for some initial insight, but nothing immediately suggested where to look. While this problem was initially reported on specific links, it appeared inconsistently during demonstrations. Sometimes it would present and not present on the same device, browser, and link. I quickly realized that controlling the variables was necessary. Enter Selenium, the Python web browsing library. By standardizing the browser environment, I aimed for consistent replication.

Replicating my browser conditions required significant experimentation, but was ultimately successful. Since the bug occurred in my personal browser for the same link, user behavior became a suspect. I tested different browser conditions, and eventually discovered that certain window dimensions reproduced the bug. I then re-read through the JavaScript code, and instantly realized that it did not address all window dimensions users could have. With the root cause identified, a solution was well in view.

The final step was to get the code fixed. While a particular solution came to mind, I didn't want to my ticket to restrict the creativity of the developer. The ticket was written to identify the desired behavior of the code without specifying the implementation of said behavior.  In addition to the ticket for an immediate fix, I wrote a secondary one to establish testing criteria to ensure that this would not happen again. For additional context, I referenced the commit that had caused the code to deviate from the expected behavior. 

This endeavor was ultimately successful in achieving our business goals. Another developer would go on to implemented the fix and its accompanying test. After it was deployed, I re-ran my SQL query to identify the incidence of malformed data. The results indicated a 0% incidence after our work was in production.

**Lessons Learned:**

- **Replication is Key:** Reproducing a software issue defines the conditions that generate it. Without understanding these conditions, you can't understand the problem itself. For web development, libraries like Selenium are invaluable for replicating real-world user experiences.
- **The Power of Monitoring:** Quantifying a software defect's prevalence is essential for triage. Even if observed, a defect might not warrant immediate attention. Monitoring also allows for measuring the impact of a resolution.
- **Behavior-Driven Development:** When writing a ticket, Behavior-Driven Development (BDD) offers valuable perspective. While there might be a temptation to prescribe an exact solution, it's too early to commit to a specific implementation. This way, you allow for more creative solutions and a more concise ticket.
