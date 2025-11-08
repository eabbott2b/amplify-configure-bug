<template>
  <div class="container">
    <h2>Amplify Re-configure Bug (Stale ClientId on Refresh)</h2>
    
    <fieldset>
      <legend>Prerequisites (Edit in Code)</legend>
      <p>REGION: <strong>{{ REGION }}</strong></p>
      <p>USER_POOL_ID: <strong>{{ USER_POOL_ID }}</strong></p>
      <p>CLIENT_ID_A (Stale): <strong>{{ CLIENT_ID_A }}</strong></p>
      <p>CLIENT_ID_B (Active): <strong>{{ CLIENT_ID_B }}</strong></p>
    </fieldset>
    
    <fieldset>
      <legend>Test User</legend>
      <input v-model="username" placeholder="Username" />
      <input v-model="verificationCode" type="password" placeholder="Password" />
    </fieldset>

    <hr>
    
    <button @click="configureClientA" class="btn">
      Step 1: Configure with Client A
    </button>
    <br><br>
    
    <button @click="configureClientB_and_SignIn" class="btn">
      Step 2: Configure with Client B & Sign In
    </button>
    <br><br>

    <button @click="forceRefresh" class="btn btn-danger">
      Step 3: Force Refresh (This will fail)
    </button>

    <hr>
    <h3>Log</h3>
    <pre>{{ log }}</pre>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import { Amplify } from 'aws-amplify';
import { signIn, fetchAuthSession, confirmSignIn, type ConfirmSignInInput, type SignInOutput, type ConfirmSignInOutput } from 'aws-amplify/auth';

// --- 1. SET YOUR PREREQUISITES HERE ---
const REGION = 'your-region'; // e.g., 'us-east-1'
const USER_POOL_ID = 'your-user-pool-id';
const CLIENT_ID_A = 'your-first-app-client-id';  // The "stale" client
const CLIENT_ID_B = 'your-second-app-client-id'; // The "active" client
// -------------------------------------

const username = ref('test-user');
const verificationCode = ref('TestPassword123!');
const log = ref('');

const logMsg = (msg: string) => {
  log.value += `[${new Date().toLocaleTimeString()}] ${msg}\n`;
};

/**
 * STEP 1: Configure Amplify with Client A.
 * This sets the internal TokenOrchestrator's config.
 */
function configureClientA() {
  const configA = {
    Auth: {
      Cognito: {
        userPoolId: USER_POOL_ID,
        userPoolClientId: CLIENT_ID_A,
        region: REGION,
      },
    },
  };
  Amplify.configure(configA);
  logMsg(`Configured with CLIENT_ID_A: ${CLIENT_ID_A}`);
}

/**
 * STEP 2: Re-configure with Client B and sign in.
 * This proves that signIn() correctly uses the new config.
 */
async function configureClientB_and_SignIn() {
  const configB = {
    Auth: {
      Cognito: {
        userPoolId: USER_POOL_ID,
        userPoolClientId: CLIENT_ID_B,
        region: REGION,
      },
    },
  };
  Amplify.configure(configB);
  logMsg(`Configured with CLIENT_ID_B: ${CLIENT_ID_B}`);


    let signInOutput: SignInOutput | undefined
    let confirmSignInOutput: ConfirmSignInOutput | undefined

    logMsg('Attempting signIn with CLIENT_ID_B...');
    try {
      signInOutput = await signIn({
        username: username.value,
        options: {
          authFlowType: 'CUSTOM_WITHOUT_SRP'
        }
      })
    } catch (err) {
      console.info('Sign in error:', err)
      return
    }

    if (signInOutput?.nextStep?.signInStep === 'CONFIRM_SIGN_IN_WITH_CUSTOM_CHALLENGE') {
      let confirmSignInInput: ConfirmSignInInput = { challengeResponse: verificationCode.value }
      try {
        confirmSignInOutput = await confirmSignIn(confirmSignInInput)
      } catch (err) {
        console.info('Confirm sign in error:', err)
      }
    }

    if (signInOutput?.isSignedIn || confirmSignInOutput?.isSignedIn) {
      logMsg('✅ signIn with CLIENT_ID_B SUCCEEDED.')
    } else {
      logMsg(`❌ signIn with CLIENT_ID_B FAILED.`);
    }

}

/**
 * STEP 3: Force a token refresh.
 * This will fail because the internal TokenOrchestrator
 * will use CLIENT_ID_A from Step 1.
 */
async function forceRefresh() {
  logMsg('Attempting forceRefresh...');
  logMsg('Open your Network tab to see the "GetTokensFromRefreshToken" request.');
  try {
    const session = await fetchAuthSession({ forceRefresh: true });
    if (session.tokens?.accessToken) {
      logMsg(`Access Token after refresh: ${session.tokens.accessToken}`);
    } else {
      logMsg('❌ No Access Token found after refresh.');
      logMsg('\n--- FAILURE ANALYSIS ---');
      logMsg(`The API call failed because the default TokenProvider ` +
           `sent CLIENT_ID_A ("${CLIENT_ID_A}") in its refresh request, ` +
           `but the refresh token was issued to CLIENT_ID_B ("${CLIENT_ID_B}").`);
    }
  } catch (e) {
    logMsg(`❌ fetchAuthSession FAILED: ${e}`);

  }
}
</script>

<style>
  body {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
    margin: 20px;
    background-color: #f7f7f7;
  }
  .container {
    max-width: 800px;
    margin: 0 auto;
    padding: 20px;
    background-color: #fff;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  }
  input {
    width: 200px;
    padding: 8px;
    margin-right: 10px;
    border: 1px solid #ccc;
    border-radius: 4px;
  }
  fieldset {
    border: 1px solid #ddd;
    border-radius: 4px;
    margin-bottom: 20px;
    padding: 15px;
  }
  legend {
    font-weight: bold;
    padding: 0 10px;
  }
  pre { 
    background: #eee; 
    padding: 15px; 
    border-radius: 4px;
    overflow-x: auto;
    white-space: pre-wrap;
  }
  .btn {
    padding: 10px 15px;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    background-color: #007bff;
    color: white;
    font-size: 14px;
  }
  .btn-danger {
    background-color: #dc3545;
  }
</style>