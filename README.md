You've provided the code for your login attempt and the HTML of the login page you received, along with a log indicating "Login failed or indicator not found." Let's analyze why the login might be failing and how to proceed.

Analyzing the Login Attempt

Login URL: You're using https://cqms.obayashi.co.jp/sso/UI/Login as the login URL. Based on the HTML you received, this appears to be the correct URL for the login form. The form in the HTML has fields for "メールアドレス" (email address) and "パスワード" (password), which aligns with the IDToken1 and IDToken2 parameters you're sending.

Form Parameters: You are sending the following form parameters:

IDToken1: UHR757 (likely the username field, but the HTML label says "メールアドレス")
IDToken2: Win22222 (likely the password field)
goto: ... (This is a placeholder and likely needs a specific value)
type: obayashi
encoded: true
gx_charset: UTF-8
Headers: You're sending a User-Agent, Accept, and Referer header, which is good practice.

Login Success Indicator: You're checking for the presence of "successful_login_indicator" in the HTML response.

Why Login is Likely Failing

Based on the information, here are the most probable reasons for the login failure:

Incorrect Credentials: The username (UHR757) or password (Win22222) might be incorrect. Double-check these values.

Case Sensitivity: Usernames and passwords are often case-sensitive. Ensure the case matches the required input.

Incorrect Form Field Names: While IDToken1 and IDToken2 seem to correspond to the input fields, the HTML form uses name="ID" for the email address and name="password" for the password. The server might be expecting these specific names.

Missing or Incorrect goto Parameter: The goto parameter is often used to redirect the user to a specific page after successful login. If this parameter is required and you're not providing the correct value, the login might fail. You need to inspect the login form in your browser's developer tools to see if there's a hidden input field named goto and what its value is when you attempt to log in manually.

CSRF Token (__RequestVerificationToken): The HTML contains a hidden input field with the name __RequestVerificationToken. This is a common security measure to prevent Cross-Site Request Forgery (CSRF) attacks. You are not currently sending this token in your form_params. The server likely expects this token to be present and valid for the login request to succeed.

JavaScript Dependency: The login process might rely on JavaScript to hash the password, set cookies, or manipulate form data before submission. If this is the case, Guzzle (which doesn't execute JavaScript) won't be able to perform the login correctly.

How to Login Correctly (Step-by-Step)

Inspect the Login Form in Your Browser:

Open the login page (https://cqms.obayashi.co.jp/sso/UI/Login) in Microsoft Edge.
Open your browser's Developer Tools (usually by pressing F12).
Go to the "Network" tab.
Attempt to log in manually using the correct credentials.
Look at the "Request" details for the POST request that is sent when you click the "ログイン" (Login) button.
Identify the exact names of all the form fields being submitted, including hidden fields like __RequestVerificationToken and goto.
Note the values of these fields. The __RequestVerificationToken will be different on each page load.
Examine the "Headers" of the successful login request to see if there are any specific headers the server is expecting.
Update Your Guzzle Code:

PHP

$client = new \GuzzleHttp\Client([
    'allow_redirects' => true,
    'cookies' => true, // Maintain session
]);

$loginUrl = 'https://cqms.obayashi.co.jp/sso/UI/Login';

try {
    // Step 1: Get the CSRF Token
    $response = $client->get($loginUrl, [
        'headers' => [
            'User-Agent' => 'Mozilla/5.0',
            'Accept' => 'text/html,application/xhtml+xml',
        ],
    ]);
    $html = (string) $response->getBody();
    $crawler = new \Symfony\Component\DomCrawler\Crawler($html);
    $csrfToken = $crawler->filter('input[name="__RequestVerificationToken"]')->attr('value');

    // Step 2: Login with Correct Parameters
    $loginResponse = $client->post($loginUrl, [
        'headers' => [
            'User-Agent' => 'Mozilla/5.0',
            'Accept' => 'text/html,application/xhtml+xml',
            'Referer' => 'https://cqms.obayashi.co.jp/',
        ],
        'form_params' => [
            'ID' => 'UHR757', // Use the 'name' attribute from the HTML
            'password' => 'Win22222', // Use the 'name' attribute from the HTML
            '__RequestVerificationToken' => $csrfToken, // Include the CSRF token
            'goto' => 'YOUR_ACTUAL_GOTO_VALUE', // Replace with the correct 'goto' value
            'type' => 'obayashi',
            'encoded' => 'true',
            'gx_charset' => 'UTF-8',
        ],
    ]);

    log::info(json_encode($loginResponse));
    $html = (string) $loginResponse->getBody();
    log::info("Full HTML after login attempt: " . $html);

    if (strpos($html, 'successful_login_indicator') !== false) {
        log::info("Login successful!");
        // Proceed to the next steps of navigating to the email search system
    } else {
        log::error("Login failed or indicator not found.");
    }

} catch (\Exception $e) {
    log::error("Login error: " . $e->getMessage());
}
Adjust the Login Success Indicator: The string "successful_login_indicator" is a placeholder. You need to examine the HTML of a successful login in your browser to find a unique element or text that indicates you are logged in. Use this as your success check in the strpos() function.

Handle Redirects: Your Guzzle client is configured to allow_redirects, which is good. Ensure that the successful login response redirects you to the next page.

Is the Login URL Correct?

Based on the HTML, the login URL https://cqms.obayashi.co.jp/sso/UI/Login appears to be the correct endpoint for submitting the login form. However, it's crucial to verify this by inspecting the form's action attribute in the browser's developer tools.

Next Steps After Successful Login:

Once you can successfully log in, the next steps will involve:

Navigating to the "メールアドレス検索システム" (Email Address Search System) as described in the PowerPoint. This might involve making additional GET requests to specific URLs. You'll need to identify these URLs by manually navigating the site after logging in.
Simulating the search by identifying the search form, its fields, and submitting the appropriate data.
Triggering the "CSV出力" (CSV Export), which likely involves finding a link or submitting another form.
Downloading the CSV file using Guzzle.
By carefully inspecting the website's behavior in your browser, you can replicate the necessary steps in your Laravel application using Guzzle. Remember to handle cookies to maintain your session across requests.
