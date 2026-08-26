<script>
  export let onLoginSuccess = (user, token) => {};
  export let isQrMode = false;

  let username = "";
  let password = "";
  let showPassword = false;
  let errorMessage = "";
  let isLoading = false;

  async function handleSubmit() {
    errorMessage = "";
    if (!username.trim() || !password.trim()) {
      errorMessage = "Please enter both username and password.";
      return;
    }

    isLoading = true;

    try {
      const res = await fetch("/api/auth/login", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ username, password })
      });

      const data = await res.json();
      isLoading = false;

      if (data.success && data.token) {
        localStorage.setItem("sdx_token", data.token);
        onLoginSuccess(data.user, data.token);
      } else {
        errorMessage = data.message || "Invalid username or password.";
      }
    } catch (err) {
      isLoading = false;
      errorMessage = "Unable to connect to authentication server. Ensure backend server is running.";
    }
  }
</script>

<div class="min-h-screen w-screen bg-slate-50 dark:bg-slate-950 text-slate-800 dark:text-slate-100 flex items-center justify-center p-4 box-border transition-colors duration-300">
  <div class="w-full max-w-md bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-800 rounded-3xl p-8 space-y-6 shadow-2xl backdrop-blur-xl">
    
    <!-- BRANDING HEADER (UNIFIED) -->
    <div class="text-center space-y-2">
      <div class="h-12 w-12 rounded-2xl bg-[#1a2b6c] border border-rose-500/30 flex items-center justify-center font-bold text-xl text-white shadow-lg mx-auto mb-2">
        <svg class="w-6 h-6 fill-current text-white" viewBox="0 0 24 24">
          <path d="M19 3H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zm-5 14H7v-2h7v2zm3-4H7v-2h10v2zm0-4H7V7h10v2z"/>
        </svg>
      </div>
      <h1 class="text-2xl font-black text-[#1a2b6c] dark:text-white tracking-tight">
        Online Survey App
      </h1>
      <p class="text-xs text-slate-500 dark:text-slate-400">
        Sign in to access your feedback dashboard and operations
      </p>
    </div>

    {#if errorMessage}
      <div class="bg-rose-50 dark:bg-rose-950/80 border border-rose-200 dark:border-rose-800 text-rose-700 dark:text-rose-300 text-xs px-4 py-3 rounded-xl text-center font-medium">
        ⚠️ {errorMessage}
      </div>
    {/if}

    <form on:submit|preventDefault={handleSubmit} class="space-y-4">
      <div class="space-y-1">
        <label for="username-input" class="text-xs font-bold text-slate-500 dark:text-slate-400 uppercase tracking-wider block">Username</label>
        <input
          id="username-input"
          type="text"
          bind:value={username}
          placeholder="Enter username or operator ID"
          class="w-full bg-slate-50 dark:bg-slate-950 border border-slate-200 dark:border-slate-800 text-[#1a2b6c] dark:text-white placeholder-slate-400 dark:placeholder-slate-600 rounded-xl px-4 py-3 text-sm focus:outline-none focus:border-[#e31b23] transition-all font-mono"
        />
      </div>

      <div class="space-y-1">
        <label for="password-input" class="text-xs font-bold text-slate-500 dark:text-slate-400 uppercase tracking-wider block">Password</label>
        <div class="relative flex items-center">
          <input
            id="password-input"
            type={showPassword ? "text" : "password"}
            bind:value={password}
            placeholder="••••••••"
            class="w-full bg-slate-50 dark:bg-slate-950 border border-slate-200 dark:border-slate-800 text-[#1a2b6c] dark:text-white placeholder-slate-400 dark:placeholder-slate-600 rounded-xl pl-4 pr-11 py-3 text-sm focus:outline-none focus:border-[#e31b23] transition-all font-mono"
          />
          <button
            type="button"
            on:click={() => (showPassword = !showPassword)}
            class="absolute right-3 p-1 text-slate-400 hover:text-slate-600 dark:hover:text-slate-200 focus:outline-none cursor-pointer"
            title={showPassword ? "Hide password" : "Show password"}
          >
            {#if showPassword}
              <!-- EYE OFF SVG -->
              <svg class="w-5 h-5 fill-current" viewBox="0 0 24 24">
                <path d="M12 7c2.76 0 5 2.24 5 5 0 .65-.13 1.26-.36 1.83l2.92 2.92c1.51-1.26 2.7-2.89 3.44-4.75-1.73-4.39-6-7.5-11-7.5-1.4 0-2.74.25-3.98.7l2.16 2.16C10.74 7.13 11.35 7 12 7zM2 4.27l2.28 2.28.46.46C3.08 8.3 1.78 10.02 1 12c1.73 4.39 6 7.5 11 7.5 1.55 0 3.03-.3 4.38-.84l.42.42L19.73 22 21 20.73 3.27 3 2 4.27zM7.53 9.8l1.55 1.55c-.05.21-.08.43-.08.65 0 1.66 1.34 3 3 3 .22 0 .44-.03.65-.08l1.55 1.55c-.67.33-1.41.53-2.2.53-2.76 0-5-2.24-5-5 0-.79.2-1.53.53-2.2zm4.31-.78l3.15 3.15.02-.17c0-1.66-1.34-3-3-3l-.17.02z"/>
              </svg>
            {:else}
              <!-- EYE ON SVG -->
              <svg class="w-5 h-5 fill-current" viewBox="0 0 24 24">
                <path d="M12 4.5C7 4.5 2.73 7.61 1 12c1.73 4.39 6 7.5 11 7.5s9.27-3.11 11-7.5c-1.73-4.39-6-7.5-11-7.5zM12 17c-2.76 0-5-2.24-5-5s2.24-5 5-5 5 2.24 5 5-2.24 5-5 5zm0-8c-1.66 0-3 1.34-3 3s1.34 3 3 3 3-1.34 3-3-1.34-3-3-3z"/>
              </svg>
            {/if}
          </button>
        </div>
      </div>

      <button
        type="submit"
        disabled={isLoading}
        class="w-full bg-[#1a2b6c] hover:bg-[#e31b23] text-white font-bold py-3.5 px-4 rounded-xl text-sm transition-all shadow-lg active:scale-[0.98] disabled:opacity-50 mt-2 cursor-pointer"
        style="color: #ffffff !important;"
      >
        {isLoading ? "Authenticating..." : "Sign In ➔"}
      </button>
    </form>

    <!-- ADMIN CONTACT NOTICE -->
    <div class="pt-4 border-t border-slate-200 dark:border-slate-800/80 text-center space-y-1">
      <p class="text-[11px] text-slate-400 dark:text-slate-500">
        Need operator or location credentials?
      </p>
      <p class="text-xs text-slate-500 dark:text-slate-400 font-medium">
        Contact your <span class="text-[#e31b23] dark:text-rose-400 font-bold">System Administrator</span> to request access.
      </p>
    </div>

  </div>
</div>