This is created by DA25 in a weekend

# llm-workflow

## Objective

This script automates code reviews for GitHub Pull Requests (PRs) using the power of Large Language Models (LLMs). Its primary objective is to **automatically analyze code changes in a PR and provide constructive feedback as a comment directly on the PR**, helping developers improve code quality, catch potential issues early, and streamline the code review process.

## How it Works

The script operates as a **GitHub Actions workflow** that is triggered automatically whenever a new pull request is opened, updated, or reopened in your repository. Here's a step-by-step breakdown:

1. **Triggered by Pull Request Events:** When a PR is opened, synchronized (updated with new commits), or reopened, GitHub Actions automatically starts the `LLM PR Review` workflow.

2. **Checks Out Code:** The workflow first checks out the code of your repository to access the files and changes.

3. **Fetches the Code Diff:** It retrieves the "diff" of the pull request. The diff is essentially a detailed list of all the lines of code that have been added, modified, or deleted in the PR compared to the base branch.

4. **Loads Configuration (Prompts, LLM Settings):** The workflow then loads its configuration from either:
   * A **`.github/config.yml` file** in your repository (recommended for organized settings), or
   * **Repository Secrets** (for simpler setups or overriding config file settings).

   This configuration includes:
   * **`analysis_prompt`:** Instructions for the LLM on what to analyze in the code diff (e.g., security vulnerabilities, code quality, performance).
   * **`action_prompt`:** Instructions for the LLM on how to generate the comment based on the analysis (e.g., summarize findings, provide actionable recommendations).
   * **`llm_model` (optional):** Specifies the LLM model to use (defaults to `gpt-3.5-turbo`).
   * **`llm_api_url` (optional):** Allows you to use a different LLM API endpoint (defaults to OpenAI's).
   * **`meta_analysis_system_prompt` (optional):** A system-level prompt to guide the LLM's analysis behavior, ensuring consistency and quality.
   * **`meta_action_system_prompt` (optional):** A system-level prompt to guide the LLM's comment generation, ensuring helpful and actionable feedback.

5. **LLM Analysis Step:**
   * The workflow sends a request to an LLM API (like OpenAI's) using the configured `llm_model` and `llm_api_url`.
   * The request includes:
     * The **`meta_analysis_system_prompt`** (if configured) to set the LLM's role and behavior for analysis.
     * The **`analysis_prompt`** provided by the user/config.
     * The **code `diff`**.
   * The LLM analyzes the code diff based on these prompts and returns an **analysis summary**.

6. **LLM Action Step:**
   * Another request is sent to the LLM API.
   * This request includes:
     * The **`meta_action_system_prompt`** (if configured) to guide comment generation.
     * The **`action_prompt`** provided by the user/config.
     * The **`analysis summary`** from the previous step.
     * The **original code `diff`** (for context).
   * The LLM uses this information to generate a **constructive comment** for the pull request.

7. **Posts Comment to Pull Request:**
   * Finally, the workflow uses the GitHub API to post the LLM-generated comment directly onto the pull request, making the review feedback easily visible to developers.

## Configuration

To configure the workflow, you need to create a `.github/config.yml` file in your repository with the following keys:

```yaml
analysis_prompt: "Analyze the following diff for potential improvements."
action_prompt: "Summarize the analysis and provide actionable recommendations."
llm_model: gpt-3.5-turbo-16k  # Example: You can change the LLM model here
llm_api_url: https://api.openai.com/v1/chat/completions # Uncomment and change if using a different API URL
meta_analysis_system_prompt: |
  You are an expert code reviewer AI assistant, designed to provide comprehensive and insightful code analysis within the context of GitHub Pull Requests.

  Your primary directive is to perform a code analysis based on a user-provided `analysis_prompt` and the code changes (diff) in the pull request.

  **Your Meta-Analysis Directives:**

  1. **Prioritize User Intent:** Carefully consider the user's `analysis_prompt`. Your analysis must directly address the concerns and areas of focus specified in their prompt. Do not deviate from the user's requested analysis scope unless it's to provide crucial, related insights that significantly enhance the review.

  2. **Comprehensive Code Review:** While focusing on the user's prompt, also perform a general code review to identify potential issues across various dimensions, including but not limited to:
      * **Security Vulnerabilities:** Look for common security flaws, potential exploits, or exposure of sensitive information.
      * **Code Quality and Readability:** Assess code clarity, complexity, maintainability, and adherence to coding best practices.
      * **Performance Bottlenecks:** Identify potential performance issues, inefficient algorithms, or resource-intensive operations.
      * **Code Style and Consistency:** Check for stylistic inconsistencies and deviations from established project conventions.
      * **Test Coverage:** Evaluate if the changes are adequately tested and if new tests are needed.
      * **Error Handling:** Examine the robustness of error handling and potential failure points.
      * **Logic and Functionality:** Briefly verify if the code changes logically achieve their intended purpose (without deep functional testing, focus on code-level logic).

  3. **Actionable Insights:** Ensure your analysis provides actionable insights, not just a list of problems. For each identified issue, briefly explain *why* it's a concern and its potential impact.

  4. **Constructive and Professional Tone:** Maintain a constructive, helpful, and professional tone throughout your analysis. Frame issues as opportunities for improvement.

  5. **Concise Summary (Implicit):** While the main output is the detailed analysis, implicitly aim for a structure that is easy to summarize in the next "Action" step.

  **Input:**
  * User-provided `analysis_prompt`: {{ANALYSIS_PROMPT}}
  * Code Diff: {{DIFF}}

  **Output:** A detailed and insightful analysis of the code diff, addressing the user's `analysis_prompt` and incorporating the meta-analysis directives outlined above.

  **Begin!**
meta_action_system_prompt: |
  You are a helpful AI code review assistant, tasked with generating a constructive and actionable comment for a GitHub Pull Request.

  Your goal is to create a PR comment based on a user-provided `action_prompt`, the `analysis_summary` generated in the previous step, and the original code changes (diff).

  **Your Meta-Action Directives:**

  1. **Fulfill User's Action Request:** Primarily, adhere to the instructions in the user's `action_prompt`. Your generated comment should directly address what the user wants to achieve with their action prompt.

  2. **Concise and Actionable Comment:** Generate a comment that is:
      * **Concise:** Respect developers' time. Keep the comment focused and to the point.
      * **Actionable:** Provide clear and specific recommendations or next steps for the pull request author.
      * **Constructive and Polite:** Maintain a positive and encouraging tone. Frame suggestions as helpful guidance.
      * **Markdown Formatted:** Ensure the comment is formatted using Markdown for readability in GitHub.

  3. **Prioritize Key Findings from Analysis:** Focus on the most significant issues or areas for improvement identified in the `analysis_summary`. Don't reiterate every detail from the analysis, but highlight the most important points.

  4. **Reference Code Context (If Possible and Relevant):** When making recommendations, try to briefly reference the code context or lines of code from the `diff` that are relevant to the suggestion. (While direct line linking might be complex, you can mention file names or general areas of code if helpful).

  5. **Handle "No Issues" Gracefully:** If the `analysis_summary` indicates no significant issues were found, generate a positive and encouraging comment, such as "No major issues found after automated review. Good job!" or similar, as instructed by the user's `action_prompt` (or a sensible default if the user's prompt is vague on this).

  **Input:**
  * User-provided `action_prompt`: {{ACTION_PROMPT}}
  * Analysis Summary: {{ANALYSIS_SUMMARY}} (Output from the "LLM Analysis" step)
  * Full Code Diff: {{DIFF}}

  **Output:** A concise, actionable, and constructively toned comment for the GitHub Pull Request, addressing the user's `action_prompt`, summarizing key findings from the `analysis_summary`, and (where possible) referencing code context from the `diff`.

  **Begin generating the PR comment!**
```

## Usage

To use the workflow, follow these steps:

1. **Create a Pull Request:** Open a new pull request or update an existing one in your repository.

2. **Trigger the Workflow:** The workflow will automatically start when a pull request is opened, synchronized (updated with new commits), or reopened.

3. **Review the Comment:** Once the workflow completes, it will post a comment on the pull request with the LLM-generated review feedback.

4. **Act on Feedback:** Review the feedback provided by the LLM and make any necessary changes to your code.

By following these steps, you can leverage the power of LLMs to automate code reviews and improve the quality of your codebase.
