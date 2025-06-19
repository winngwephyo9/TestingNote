   /* --- Loading Indicator Styles --- */
        #loader {
            position: absolute;
            left: 50%;
            top: 50%;
            z-index: 100; /* Make sure it's on top */
            width: 150px;
            height: 150px;
            margin: -75px 0 0 -75px; /* Center it */
            border: 16px solid #f3f3f3; /* Light grey */
            border-radius: 50%;
            border-top: 16px solid #3498db; /* Blue */
            width: 120px;
            height: 120px;
            -webkit-animation: spin 2s linear infinite; /* Safari */
            animation: spin 2s linear infinite;
            display: block; /* Initially visible */
        }

        #loader-text {
            position: absolute;
            left: 50%;
            top: calc(50% + 80px); /* Below the spinner */
            transform: translateX(-50%);
            color: #555;
            font-family: sans-serif;
            font-size: 16px;
            z-index: 100;
             display: block; /* Initially visible */
        }

        /* Safari */
        @-webkit-keyframes spin {
            0% { -webkit-transform: rotate(0deg); }
            100% { -webkit-transform: rotate(360deg); }
        }

        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
        
      <div id="loader"></div>
    <div id="loader-text">Loading 3D Model...</div>

    
        // --- Get Loader Elements ---
const loaderElement = document.getElementById('loader');
const loaderTextElement = document.getElementById('loader-text');


// Show loader before starting
if (loaderElement) loaderElement.style.display = 'block';
if (loaderTextElement) loaderTextElement.style.display = 'block';



// Hide loader now that everything is ready
        if (loaderElement) loaderElement.style.display = 'none';
        if (loaderTextElement) loaderTextElement.style.display = 'none';
