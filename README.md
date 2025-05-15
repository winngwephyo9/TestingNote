<?php

namespace App\Http\Controllers;

use GuzzleHttp\Client;
use Illuminate\Http\Request;

class ObayashiLoginController extends Controller
{
    public function login()
    {
        $client = new Client([
            'cookies' => true, // Enable cookie handling
            'headers' => [
                'User-Agent' => 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36',
                'Accept' => 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7',
                'Accept-Encoding' => 'gzip, deflate',
                'Accept-Language' => 'ja-JP,ja;q=0.9,en-US;q=0.8,en;q=0.7',
                'Cache-Control' => 'no-cache',
                'Connection' => 'keep-alive',
                'Origin' => 'http://login.fc.obayashi.co.jp',
                'Pragma' => 'no-cache',
                'Referer' => 'http://login.fc.obayashi.co.jp/sso/UI/Login?type=obayashi&goto=http%3A%2F%2Fintralogin.fc.obayashi.co.jp%2Fcgi-bin%2Fclogin.cgi%3Faccess%3Dhttp%253A%252F%252Fwww.fc.obayashi.co.jp%252F%252F',
                'Upgrade-Insecure-Requests' => '1',
            ],
        ]);

        $loginUrl = 'http://login.fc.obayashi.co.jp/sso/UI/Login';
        $successRedirectUrl = 'http://intralogin.fc.obayashi.co.jp/cgi-bin/clogin.cgi?access=http%3A%2F%2Fwww.fc.obayashi.co.jp%2F';

        $username = 'YOUR_USERNAME'; // Replace with the actual username
        $password = 'YOUR_PASSWORD'; // Replace with the actual password
        $groupCompanyCode = 'U'; // Replace with the desired group company code (e.g., 'U' for 大林組)

        try {
            // 1. Get the initial login page to potentially retrieve any necessary tokens or cookies
            $response = $client->get($loginUrl);
            $html = (string) $response->getBody();

            // You might need to parse the HTML here to extract any dynamic values
            // if the form has anti-CSRF tokens or other dynamic fields.
            // Based on the provided HTML, it doesn't seem to have any critical dynamic tokens.

            // 2. Prepare the login form data
            $loginData = [
                'IDToken0-0' => $groupCompanyCode,
                'IDToken0' => $groupCompanyCode, // Assuming the company code is the same as the group code initially
                'IDToken1' => $groupCompanyCode . $username, // Apply the JavaScript logic
                'IDToken2' => $password,
                'Login.Submit' => 'ログイン',
                'goto' => 'aHR0cDovL2ludHJhbG9naW4uZmMub2JheWFzaGkuY28uanAvY2dpLWJpbi9jbG9naW4uY2dpP2FjY2Vzcz1odHRwOi8vd3d3LmZjLm9iYXlhc2hpLmNvLmpwLw==',
                'gotoOnFail' => '',
                'SunQueryParamsString' => 'dHlwZT1vYmF5YXNoaQ==',
                'encoded' => 'true',
                'type' => 'obayashi',
                'gx_charset' => 'UTF-8',
                'IDButton' => 'ログイン', // Set the IDButton value as per the JavaScript
            ];

            // 3. Submit the login form
            $response = $client->post($loginUrl, [
                'form_params' => $loginData,
            ]);

            // 4. Check if the login was successful by inspecting the response and cookies
            // A successful login should result in a redirect to the success URL
            if ($response->getStatusCode() === 302 && $response->getHeaderLine('Location') === $successRedirectUrl) {
                // Optionally, you can make a subsequent request to the redirected URL
                $successPageResponse = $client->get($successRedirectUrl);
                $successPageContent = (string) $successPageResponse->getBody();

                // Do something with the successful login page content
                dd('Login successful!', $successPageContent);
            } else {
                // Login failed
                dd('Login failed.', (string) $response->getBody());
            }

        } catch (\GuzzleHttp\Exception\GuzzleException $e) {
            dd('An error occurred: ' . $e->getMessage());
        }
    }
}

![画像 (16)](https://github.com/user-attachments/assets/813622b4-a900-46cd-ab5d-ecc54c822c14)
![画像 (14)](https://github.com/user-attachments/assets/25ccd9e5-c8d6-4f79-8235-cbd264344471)
[login.txt](https://github.com/user-attachments/files/20218578/login.txt)
