---
name: web-explainer
description: Guidance for building clear, delightful web pages, especially explainers, tutorials and documentation — compact prose, purposeful design, and plain HTML/CSS/JS. Use when writing or designing a web-based explainer or docs site.
---

# Building web explainers

This is guidance for building comprehensible and delightful web experiences, particularly explainers, tutorials and documentation. Nothing here supersedes instructions from the user. Aspects of this skill may be useful for other purposes.

## Content guidelines
- Keep your target audience in mind, remember they are human with human attention spans and general knowledge distributions
- Try to make your prose as compact as possible without introducing complexity/jargon, this is very different from writing for yourself/other agents
- Keep your purpose in mind. Just because a particular topic is very important in a source doesn't mean it necessarily needs a large presence (or any at all) in what you're writing. 
- Use sub-agents to review and edit your content, for example you might dump a bunch of details into a sub-agent and have it write a line that captures the essence, or review a draft for clarity/simplicity without the existing context you have
- Tell your content sub-agents who they are (a non-technical manager, a web-dev without low-level experience, etc) as needed
- Periodically take a step back, and make sure the structure and flow of your writing is still sensible. Reorganize and edit as needed
- Use varied sentence length and structure throughout the project
- Avoid stating what *isn't* true, unless a reader is genuinely likely to have a false impression for some reason
- Always consolidate and simplify your prose if you can do so without degrading quality
- Never write more to fill up space. If a one-word "sentence" says what you want to say, that's the right choice.
- When editing, be mindful if you're adding or subtracting. Sometimes the right "correction" to a mistake is simply removing whatever is incorrect
- Ultimately, you're in charge of making the content as clear, complete and easy to understand as possible. The first human to see it might themselves be the target audience

## Graphic design
- Your primary objective is to make the experience instantly and effortlessly informative to the target audience
- Secondarily, it's to make it delightful and aesthetic
- Use subtle animations and transitions with purpose, make sure they're not detracting or make the experience sluggish
- Avoid excessive "structure" in the page. You usually don't want many layers of visually distinct containers inside each other
- Avoid thin box outlines and *never* pair a single-side border with rounded corners in a way that results in a crescent corner
- Every element on the page should have a purpose, if removing any given element does not degrade the experience, do it
- Design things adaptive to various screen sizes where practical
- Visually inspect your graphics and layout to make sure everything looks good, don't push QA onto the user
- Have a light/dark toggle that follows the browser theme by default
- Use the same design language and elements consistently, and capture them with abstractions
- Use interactivity sparingly or where asked. A (?) -> More details popup, interactive parameterized graphic or scroll-keyed animation can elevate the experience, but too much of that sort of stuff can be detracting or confusing

## Technicals
- Prefer plain HTML/CSS/JS with no compiler, framework or dependencies
- Use modern web features (templates, query selector, JS modules, etc), don't worry about compat with ancient browsers
- Prefer a single-page application where it makes sense
- You're in charge of the code, keep it top notch: refactor liberally, abstract and dedup, if you spot an opportunity to make the code better it's likely worth taking a bit longer on your current task to do that now
- You need a way to view and test your web app in a browser, the `playwright-cli` skill from `@playwright/cli@latest` is a good option
- Delegate to sub-agents for specific tasks (writing and validating an SVG, iterating on design until it looks good, researching a topic and writing up a report, etc)
