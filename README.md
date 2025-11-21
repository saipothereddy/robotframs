GitLab UI Automation Project Documentation-----------------------------------------
1. Overview
 This project automates GitLab UI workflows using Robot Framework with Selenium.
 The work completed includes:- Okta Single Sign-On (SSO) automation.- GitLab UI navigation and search.- Project selection and subgroup handling.- Validation of GitLab pages.- Screenshot capture.- Token Suite with Test Case A/B/C logic.- Failure resilience and conditional flow validation.
 2. Token Suite (token_suite.robot)
 Contains:- Acquire Token With Error Handling (Suite Setup)- Test Case A (may pass/fail based on token)- Test Case B (always runs)- Test Case C (validates rule: A fail -> B must run)
 3. Okta SSO Automation (gitlab_sso_login.robot)- Open browser via Selenium.- Navigate to GitLab login.- Enter Okta email and password.- Handle hidden password elements.- Redirect to GitLab dashboard.
4. GitLab Navigation (gitlab_navigation.robot)- Navigate to Projects.- Search project using input field.- Trigger search via ENTER key.- Select project from dropdown or list.- Handle subgroup navigation.
 5. GitLab Validation (gitlab_validation.robot)- Validate correct project page.- Validate breadcrumbs.- Validate repository landing page.- Capture screenshot for results.
 6. Full Flow Summary- Suite Setup runs and attempts token creation.- Test A executes first.- Test B always executes after A.- Test C validates A->B rule.- UI Login via Okta SSO.- GitLab project search and navigation.- UI validation and screenshot capture.
 7. XPaths Developed- Okta password input:
 //input[@type='password' and contains(@class,'okta-form-input-field')]- GitLab search field:
 //input[@placeholder='Search your projects']- Flexible project selectors:
(
 //a[.//span[contains(., '${PROJECT_NAME}')]]
 |
 //div[contains(@class,'gl-search-box-results')]//a[.//span[contains(.,'${PROJECT_NAME}')]]
 )
 8. Deliverables Completed- SSO login automation completed.- GitLab UI navigation flow automated.- Subgroup-aware project selection.- Robust validation logic.- Test A/B/C suite validation architecture.- Screenshot functionality.- Documentation delivered.
 9. References- Robot Framework: https://robotframework.org/- SeleniumLibrary: https://robotframework.org/SeleniumLibrary/SeleniumLibrary.html- Okta Developer Docs: https://developer.okta.com/- GitLab Documentation: https://docs.gitlab.com/- Microsoft Edge WebDriver: https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/- Python RequestsLibrary: https://marketsquare.github.io/robotframework-requests/- XPath Reference: https://www.w3.org/TR/xpath/all
