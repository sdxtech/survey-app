<script>
  import { onMount } from "svelte";
  import { scale } from "svelte/transition";
  import Dashboard from "./components/Dashboard.svelte";
  import FormBuilder from "./components/FormBuilder.svelte";
  import Kiosk from "./components/Kiosk.svelte";
  import Answers from "./components/Answers.svelte";
  import Login from "./components/Login.svelte";
  import DeviceManagement from "./components/DeviceManagement.svelte";
  import UserManagement from "./components/UserManagement.svelte";

  const API_BASE = "/api";

  let activeTab = "surveys";
  let surveysList = [];
  let responses = [];
  let activeSurveyId = "";
  let isOfflineMode = false;
  let isDedicatedKioskMode = false;
  let isSidebarExpanded = false;
  let isDarkMode = true;

  // GLOBAL DUAL-APP ENGINE MODE (PERSISTED IN LOCALSTORAGE)
  let isQrMode = false;

  // SHARE HUB MODAL STATE
  let activeShareSurvey = null;
  let showShareModal = false;
  let copyBannerMessage = "";

  // AUTHENTICATION STATE
  let currentUser = null;
  let isAuthChecking = true;

  function toggleSidebar() {
    isSidebarExpanded = !isSidebarExpanded;
    localStorage.setItem('sdx_sidebar_expanded', isSidebarExpanded ? 'true' : 'false');
  }

  function toggleTheme() {
    isDarkMode = !isDarkMode;
    if (isDarkMode) {
      document.documentElement.classList.add('dark');
      localStorage.setItem('sdx_theme', 'dark');
    } else {
      document.documentElement.classList.remove('dark');
      localStorage.setItem('sdx_theme', 'light');
    }
  }

  function handleKeyDown(event) {
    if ((event.ctrlKey || event.metaKey) && event.key.toLowerCase() === 'm') {
      event.preventDefault();
      if (currentUser?.role === 'admin') {
        toggleAppMode();
      }
    }
  }

  async function toggleAppMode() {
    if (currentUser && currentUser.role !== 'admin') return;
    isQrMode = !isQrMode;
    localStorage.setItem("sdx_app_mode", isQrMode ? "qr" : "kiosk");
    await refreshDataLedger();
  }

  function switchTab(tab) {
    const isOperator = currentUser && (currentUser.role === "kiosk_operator" || currentUser.role === "user");
    const isSiteLeader = currentUser && currentUser.role === "site_leader";

    if (isOperator || isSiteLeader) {
      activeTab = "answers";
      return;
    }

    if (currentUser && currentUser.role !== "admin" && (tab === "surveys" || tab === "builder" || tab === "devices" || tab === "users" || tab === "kiosk")) {
      activeTab = "answers";
      return;
    }
    
    if (tab === "kiosk" && !isDedicatedKioskMode) {
      activeSurveyId = "";
    }

    activeTab = tab;

    if (!isDedicatedKioskMode) {
      window.location.hash = `/${tab}`;
    }

    if (tab === "surveys" || tab === "answers") {
      setTimeout(() => {
        refreshDataLedger();
      }, 0);
    }
  }

  $: if (activeSurveyId) {
    localStorage.setItem("sdx_active_survey_id", activeSurveyId);
  }

  function normalizeSurvey(s) {
    if (!s) return s;
    return {
      ...s,
      appMode: s.appMode || "kiosk",
      assignedSite: s.assignedSite || "",
      pinCode: s.pinCode || "1234",
      thankYouMessage: s.thankYouMessage || "Thank you for your feedback! This screen will automatically refresh in a few seconds.",
      autoRefreshSeconds: s.autoRefreshSeconds !== undefined && s.autoRefreshSeconds !== null ? Number(s.autoRefreshSeconds) : 4,
      questions: (s.questions || []).map((q) => ({
        ...q,
        questionImage: q.questionImage || "",
        isRequired: Boolean(q.isRequired),
        allowMultiple: Boolean(q.allowMultiple),
        enableOptionImages: Boolean(q.enableOptionImages),
        enableOtherOption: Boolean(q.enableOtherOption),
        options: q.options || [],
        optionImages: q.optionImages || {},
        alertTriggerValues: q.alertTriggerValues || []
      })),
    };
  }

  async function registerDeviceHeartbeat() {
    const savedDeviceId = localStorage.getItem("sdx_device_id") || "Tablet-Client";
    try {
      await fetch(`${API_BASE}/devices/heartbeat`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          deviceName: savedDeviceId,
          loggedInUser: currentUser?.username || "Guest Operator",
          pairedSurveyId: activeSurveyId || null
        })
      });
    } catch (err) {}
  }

  onMount(async () => {
    window.addEventListener('keydown', handleKeyDown);

    window.addEventListener('popstate', () => {
      const hash = window.location.hash;
      const route = hash.replace("#/", "").split("?")[0];
      if (["surveys", "builder", "kiosk", "answers", "devices", "users"].includes(route)) {
        activeTab = route;
      }
    });

    const savedTheme = localStorage.getItem('sdx_theme');
    if (savedTheme === 'light') {
      isDarkMode = false;
      document.documentElement.classList.remove('dark');
    } else {
      isDarkMode = true;
      document.documentElement.classList.add('dark');
    }

    const savedSidebar = localStorage.getItem('sdx_sidebar_expanded');
    isSidebarExpanded = savedSidebar === 'true';

    const searchParams = new URLSearchParams(window.location.search);
    const hashPart = window.location.hash;
    const hashParams = new URLSearchParams(hashPart.includes("?") ? hashPart.split("?")[1] : "");

    const urlSurveyId = searchParams.get("id") || hashParams.get("id");
    const modeParam = searchParams.get("mode") || hashParams.get("mode");
    const siteParam = searchParams.get("site") || hashParams.get("site");

    if (siteParam) {
      localStorage.setItem("sdx_device_id", siteParam.trim());
    }

    if (urlSurveyId) {
      activeSurveyId = urlSurveyId;
    }

    if (urlSurveyId || modeParam || hashPart.includes("kiosk")) {
      if (modeParam === 'qr') {
        isQrMode = true;
        localStorage.setItem("sdx_app_mode", "qr");
      } else if (modeParam === 'kiosk') {
        isQrMode = false;
        localStorage.setItem("sdx_app_mode", "kiosk");
      }
      activeTab = "kiosk";
      isDedicatedKioskMode = true;
      isSidebarExpanded = false;
      isAuthChecking = false;
      refreshDataLedger();
      return;
    }

    const storedToken = localStorage.getItem("sdx_token");
    if (storedToken) {
      try {
        const res = await fetch(`${API_BASE}/auth/me`, {
          headers: { Authorization: `Bearer ${storedToken}` }
        });
        const data = await res.json();
        if (data.success && data.user) {
          currentUser = data.user;
          if (currentUser.role === "site_leader") {
            isQrMode = true;
            localStorage.setItem("sdx_app_mode", "qr");
            activeTab = "answers";
          } else if (currentUser.role === "kiosk_operator" || currentUser.role === "user") {
            isQrMode = false;
            localStorage.setItem("sdx_app_mode", "kiosk");
            activeTab = "answers";
          } else if (currentUser.role !== "admin") {
            activeTab = "answers";
          } else {
            const savedAppMode = localStorage.getItem("sdx_app_mode");
            isQrMode = savedAppMode === "qr";
            const route = hashPart.replace("#/", "").split("?")[0];
            if (["surveys", "builder", "kiosk", "answers", "devices", "users"].includes(route)) {
              activeTab = route;
            } else {
              window.location.hash = "/surveys";
            }
          }
        } else {
          localStorage.removeItem("sdx_token");
        }
      } catch (err) {
        console.warn("Session validation error:", err);
      }
    }

    isAuthChecking = false;
    await refreshDataLedger();
    registerDeviceHeartbeat();
  });

  function handleLoginSuccess(user, token) {
    currentUser = user;
    if (currentUser.role === "site_leader") {
      isQrMode = true;
      localStorage.setItem("sdx_app_mode", "qr");
      switchTab("answers");
    } else if (currentUser.role === "kiosk_operator" || currentUser.role === "user") {
      isQrMode = false;
      localStorage.setItem("sdx_app_mode", "kiosk");
      switchTab("answers");
    } else if (currentUser.role !== "admin") {
      switchTab("answers");
    } else {
      switchTab("surveys");
    }
    refreshDataLedger();
    registerDeviceHeartbeat();
  }

  function handleLogout() {
    localStorage.removeItem("sdx_token");
    localStorage.removeItem("sdx_active_survey_id");
    currentUser = null;
  }

  async function refreshDataLedger() {
    try {
      const searchParams = new URLSearchParams(window.location.search);
      const hashPart = window.location.hash;
      const hashParams = new URLSearchParams(hashPart.includes("?") ? hashPart.split("?")[1] : "");
      const urlSurveyId = searchParams.get("id") || hashParams.get("id");

      const currentModeQuery = isQrMode ? "qr" : "kiosk";
      const surveyRes = await fetch(`${API_BASE}/surveys?mode=${currentModeQuery}`);
      if (surveyRes.ok) {
        const surveyData = await surveyRes.json();
        if (surveyData.success) {
          let rawSurveys = (surveyData.surveys || []).map(normalizeSurvey);

          if (urlSurveyId && !rawSurveys.some(s => String(s._id) === String(urlSurveyId))) {
            try {
              const fallbackRes = await fetch(`${API_BASE}/surveys?mode=${isQrMode ? 'kiosk' : 'qr'}`);
              const fallbackData = await fallbackRes.json();
              if (fallbackData.success && Array.isArray(fallbackData.surveys)) {
                rawSurveys = [...rawSurveys, ...fallbackData.surveys.map(normalizeSurvey)];
              }
            } catch (e) {}
          }

          surveysList = rawSurveys;
          isOfflineMode = false;

          if (urlSurveyId && surveysList.some(s => String(s._id) === String(urlSurveyId))) {
            activeSurveyId = urlSurveyId;
            const targetSurveyObj = surveysList.find(s => String(s._id) === String(urlSurveyId));
            if (targetSurveyObj && targetSurveyObj.appMode === 'kiosk' && !currentUser) {
              isQrMode = false;
            }
          } else if (!activeSurveyId && surveysList.length > 0 && activeTab !== "kiosk") {
            activeSurveyId = surveysList[0]._id;
          }
        }
      }

      if (!isDedicatedKioskMode && activeTab !== "kiosk") {
        const responseRes = await fetch(`${API_BASE}/responses?mode=${currentModeQuery}`);
        if (responseRes.ok) {
          const responseData = await responseRes.json();
          if (responseData.success && responseData.responses) {
            responses = responseData.responses;
          }
        }
      }
    } catch (err) {
      console.warn("API link delay:", err);
      isOfflineMode = true;
    }
  }

  $: activeSurveyTitles = surveysList.map((s) => s.title);
  $: validResponsesCount = responses.filter((r) =>
    activeSurveyTitles.includes(r.surveyTitle),
  ).length;

  $: activeSurvey = surveysList.find((s) => s._id === activeSurveyId) || {
    title: "",
    questions: [],
    pinCode: "1234",
    thankYouMessage: "Thank you for your feedback! This screen will automatically refresh in a few seconds.",
    autoRefreshSeconds: 4,
    assignedSite: ""
  };

  function handleCreateNewSurvey() {
    if (currentUser?.role !== "admin") return;
    const draftId = `DRAFT-${Date.now()}`;
    const newDraftSurvey = {
      _id: draftId,
      title: isQrMode ? "New Web QR Form Schema" : "New Kiosk Terminal Schema",
      appMode: isQrMode ? "qr" : "kiosk",
      assignedSite: "",
      pinCode: "1234",
      thankYouMessage: "Thank you for your feedback! This screen will automatically refresh in a few seconds.",
      autoRefreshSeconds: 4,
      questions: [],
      isDraft: true,
    };

    surveysList = [...surveysList, newDraftSurvey];
    activeSurveyId = draftId;
    switchTab("builder");
  }

  async function persistActiveSurveyState(updatedTitle, updatedQuestions, updatedThankYouMessage, updatedAutoRefreshSeconds, assignedSiteParam) {
    if (!activeSurveyId || currentUser?.role !== "admin") return;

    const questionsToSave = typeof structuredClone === 'function'
      ? structuredClone(updatedQuestions)
      : JSON.parse(JSON.stringify(updatedQuestions));

    const cleanSeconds = Math.max(1, Number(updatedAutoRefreshSeconds) || 4);

    const resolvedAssignedSite = typeof assignedSiteParam === 'string' 
      ? assignedSiteParam.trim() 
      : (activeSurvey.assignedSite || "").trim();

    const payload = {
      title: String(updatedTitle || '').trim(),
      appMode: isQrMode ? "qr" : "kiosk",
      assignedSite: resolvedAssignedSite,
      pinCode: activeSurvey.pinCode || "1234",
      questions: questionsToSave,
      thankYouMessage: updatedThankYouMessage !== undefined ? updatedThankYouMessage : activeSurvey.thankYouMessage,
      autoRefreshSeconds: cleanSeconds
    };

    if (String(activeSurveyId).startsWith("DRAFT-") || activeSurvey.isDraft) {
      try {
        const res = await fetch(`${API_BASE}/surveys`, {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify(payload),
        });
        const data = await res.json();
        if (data.success && data.survey) {
          const normalized = normalizeSurvey(data.survey);
          surveysList = surveysList.map((s) =>
            s._id === activeSurveyId ? normalized : s
          );
          activeSurveyId = normalized._id;
          await refreshDataLedger();
        } else {
          alert(`⚠️ Failed to create survey: ${data.error || data.message || 'Unknown error'}`);
        }
      } catch (err) {
        console.error("Error creating survey in database:", err);
        alert("⚠️ Server connection error while creating survey.");
      }
      return;
    }

    try {
      const res = await fetch(`${API_BASE}/surveys/${activeSurveyId}`, {
        method: "PUT",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(payload),
      });
      const data = await res.json();
      if (data.success && data.survey) {
        const normalized = normalizeSurvey(data.survey);
        
        surveysList = surveysList.map((s) =>
          String(s._id) === String(activeSurveyId) ? normalized : s
        );

        await refreshDataLedger();
      } else {
        alert(`⚠️ Failed to update survey: ${data.error || data.message || 'HTTP ' + res.status}`);
        await refreshDataLedger();
      }
    } catch (err) {
      console.error("Error updating survey in database:", err);
      alert("⚠️ Network or server error while updating survey.");
    }
  }

  async function handleDeleteSurvey(id) {
    if (currentUser?.role !== "admin") return;
    try {
      if (
        !String(id).startsWith("DRAFT-") &&
        !String(id).startsWith("LOCAL-")
      ) {
        await fetch(`${API_BASE}/surveys/${id}`, { method: "DELETE" });
      }
      surveysList = surveysList.filter((s) => s._id !== id);
      if (activeSurveyId === id) {
        activeSurveyId = surveysList[0]?._id || "";
      }
      await refreshDataLedger();
    } catch (err) {
      surveysList = surveysList.filter((s) => s._id !== id);
    }
  }

  async function registerResponse(formattedAnswers, explicitDeviceId) {
    try {
      const searchParams = new URLSearchParams(window.location.search);
      const hashPart = window.location.hash;
      const hashParams = new URLSearchParams(hashPart.includes("?") ? hashPart.split("?")[1] : "");
      
      const urlSiteParam = searchParams.get("site") || hashParams.get("site");
      const savedSite = localStorage.getItem("sdx_device_id");

      const resolvedSiteId = explicitDeviceId || urlSiteParam || savedSite || (isQrMode ? "Web-QR-Scan" : "Tablet-A");

      const res = await fetch(`${API_BASE}/responses`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          surveyTitle: activeSurvey.title,
          deviceId: resolvedSiteId,
          appMode: isQrMode ? "qr" : "kiosk",
          answers: formattedAnswers,
        }),
      });
      const data = await res.json();
      if (data.success) {
        await refreshDataLedger();
      }
    } catch (err) {}
  }

  function handleSelectAndEdit(id) {
    if (currentUser?.role !== "admin") return;
    activeSurveyId = id;
    switchTab("builder");
  }

  function handleSelectAndTest(id) {
    activeSurveyId = id;
    switchTab("kiosk");
  }

  function handleOpenShareModal(survey) {
    activeShareSurvey = survey;
    showShareModal = true;
    copyBannerMessage = "";
  }

  function handleCloseShareModal() {
    showShareModal = false;
    activeShareSurvey = null;
    copyBannerMessage = "";
  }

  function getKioskLink(survey, forceQr = false) {
    let host = window.location.hostname;
    if (host === 'localhost' || host === '127.0.0.1') {
      host = 'digital-survey-appp-zeta.vercel.app'; 
    } else {
      host = window.location.host;
    }
    const protocol = window.location.protocol;
    const surveyId = typeof survey === 'object' ? survey._id : survey;
    const isQrSurvey = typeof survey === 'object' ? survey.appMode === 'qr' : isQrMode;
    
    const modeString = (forceQr || isQrSurvey) ? '&mode=qr' : '&mode=kiosk';

    let targetSite = "";
    if (typeof survey === 'object' && survey.assignedSite) {
      targetSite = survey.assignedSite;
    } else if (currentUser && currentUser.assignedSite) {
      targetSite = currentUser.assignedSite;
    }

    const siteString = targetSite ? `&site=${encodeURIComponent(targetSite)}` : '';
    return `${protocol}//${host}/?id=${surveyId}${modeString}${siteString}#/kiosk`;
  }

  function copyKioskLink(survey, forceQr = false) {
    const directLink = getKioskLink(survey, forceQr);
    navigator.clipboard.writeText(directLink).then(() => {
      copyBannerMessage = "✓ Link copied to clipboard!";
      setTimeout(() => {
        copyBannerMessage = "";
      }, 3000);
    }).catch(() => {
      copyBannerMessage = "⚠️ Failed to copy link.";
    });
  }
</script>

{#if isAuthChecking}
  <div class="h-screen w-screen theme-bg-main flex items-center justify-center text-cyan-500 font-mono text-sm overflow-hidden">
    <div class="flex items-center space-x-3">
      <div class="h-3 w-3 rounded-full bg-cyan-500 animate-ping"></div>
      <span class="theme-text-primary">Loading Feedback Terminal...</span>
    </div>
  </div>

{:else if !currentUser && !isDedicatedKioskMode}
  <Login onLoginSuccess={handleLoginSuccess} {isQrMode} />

{:else}
  <div class="flex h-screen w-screen max-w-full max-h-screen theme-bg-main theme-text-primary overflow-hidden m-0 p-0 fixed inset-0">
    
    <!-- SIDEBAR -->
    {#if !isDedicatedKioskMode && activeTab !== 'kiosk'}
      <aside class="{isSidebarExpanded ? 'w-64' : 'w-20'} theme-bg-sidebar theme-border border-r flex flex-col justify-between shrink-0 h-full z-40 transition-all duration-300 overflow-hidden text-slate-100">
        <div class="flex flex-col h-full justify-between">
          <div>
            
            <div class="px-4 h-16 theme-border border-b flex items-center justify-between box-border shrink-0">
              <div class="flex items-center space-x-3 overflow-hidden">
                <button
                  on:click={toggleSidebar}
                  class="p-2.5 rounded-xl text-slate-300 hover:text-white bg-white/10 hover:bg-white/20 transition-all flex items-center justify-center focus:outline-none active:scale-95 shadow-sm shrink-0 cursor-pointer"
                  title={isSidebarExpanded ? "Collapse to Icon Rail" : "Expand Sidebar"}
                >
                  <svg class="w-5 h-5 fill-current" viewBox="0 0 24 24">
                    <path d="M3 18h18v-2H3v2zm0-5h18v-2H3v2zm0-7v2h18V6H3z"/>
                  </svg>
                </button>

                {#if isSidebarExpanded}
                  <div class="flex items-center space-x-2.5 min-w-0 truncate">
                    <div class="h-8 w-8 rounded-xl bg-[#e31b23] border border-rose-500/40 flex items-center justify-center font-extrabold text-white text-xs shadow-md shrink-0">
                      {isQrMode ? 'QR' : 'EK'}
                    </div>
                    <span class="font-black text-sm tracking-tight text-white truncate">
                      {isQrMode ? 'Web QR Hub' : 'Enterprise Kiosk'}
                    </span>
                  </div>
                {/if}
              </div>
            </div>

            <!-- NAVIGATION ITEMS (UNIFIED RED ACTIVE STATE IN BOTH MODES) -->
            <nav class="p-3 space-y-2">
              {#if currentUser?.role === "admin"}
                <button
                  class="w-full flex items-center space-x-3 px-3.5 py-3 rounded-xl font-bold text-xs transition-all cursor-pointer {activeTab === 'surveys' ? 'bg-[#e31b23] text-white shadow-md' : 'text-slate-300 hover:bg-white/10 hover:text-white'} {isSidebarExpanded ? '' : 'justify-center px-0'}"
                  on:click={() => switchTab("surveys")}
                  title={isQrMode ? 'QR Forms Hub' : 'Surveys Portal'}
                >
                  <svg class="w-5 h-5 shrink-0 fill-current" viewBox="0 0 24 24">
                    <path d="M19 3H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zm-5 14H7v-2h7v2zm3-4H7v-2h10v2zm0-4H7V7h10v2z"/>
                  </svg>
                  {#if isSidebarExpanded}
                    <span class="truncate">{isQrMode ? 'QR Forms Hub' : 'Surveys Portal'}</span>
                  {/if}
                </button>

                <button
                  class="w-full flex items-center space-x-3 px-3.5 py-3 rounded-xl font-bold text-xs transition-all cursor-pointer {activeTab === 'builder' ? 'bg-[#e31b23] text-white shadow-md' : 'text-slate-300 hover:bg-white/10 hover:text-white'} {isSidebarExpanded ? '' : 'justify-center px-0'}"
                  on:click={() => switchTab("builder")}
                  disabled={surveysList.length === 0}
                  title={isQrMode ? 'QR Form Designer' : 'Form Designer'}
                >
                  <svg class="w-5 h-5 shrink-0 fill-current {surveysList.length === 0 ? 'opacity-40' : ''}" viewBox="0 0 24 24">
                    <path d="M3 17.25V21h3.75L17.81 9.94l-3.75-3.75L3 17.25zM20.71 7.04c.39-.39.39-1.02 0-1.41l-2.34-2.34c-.39-.39-1.02-.39-1.41 0l-1.83 1.83 3.75 3.75 1.83-1.83z"/>
                  </svg>
                  {#if isSidebarExpanded}
                    <span class="truncate {surveysList.length === 0 ? 'opacity-40' : ''}">{isQrMode ? 'QR Form Designer' : 'Form Designer'}</span>
                  {/if}
                </button>
              {/if}

              {#if currentUser?.role !== "site_leader" && currentUser?.role !== "kiosk_operator" && currentUser?.role !== "user"}
                <button
                  class="w-full flex items-center space-x-3 px-3.5 py-3 rounded-xl font-bold text-xs transition-all cursor-pointer {activeTab === 'kiosk' ? 'bg-[#e31b23] text-white shadow-md' : 'text-slate-300 hover:bg-white/10 hover:text-white'} {isSidebarExpanded ? '' : 'justify-center px-0'}"
                  on:click={() => switchTab("kiosk")}
                  title={isQrMode ? "Preview QR Web Form" : "Live Kiosk Mode"}
                >
                  <svg class="w-5 h-5 shrink-0 fill-current" viewBox="0 0 24 24">
                    <path d="M17 1.01L7 1c-1.1 0-2 .9-2 2v18c0 1.1.9 2 2 2h10c1.1 0 2-.9 2-2V3c0-1.1-.9-1.99-2-1.99zM17 19H7V5h10v14z"/>
                  </svg>
                  {#if isSidebarExpanded}
                    <span class="truncate">{isQrMode ? 'Preview QR Form' : 'Live Kiosk Mode'}</span>
                  {/if}
                </button>
              {/if}

              <button
                class="w-full flex items-center space-x-3 px-3.5 py-3 rounded-xl font-bold text-xs transition-all cursor-pointer {activeTab === 'answers' ? 'bg-[#e31b23] text-white shadow-md' : 'text-slate-300 hover:bg-white/10 hover:text-white'} {isSidebarExpanded ? '' : 'justify-center px-0'}"
                on:click={() => switchTab("answers")}
                title="Answers Log"
              >
                <svg class="w-5 h-5 shrink-0 fill-current" viewBox="0 0 24 24">
                  <path d="M19 3H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zm-2 10h-4v4h-2v-4H7v-2h4V7h2v4h4v2z"/>
                </svg>
                {#if isSidebarExpanded}
                  <span class="truncate">
                    {isQrMode ? 'QR Answers Log' : 'Answers Log'}
                  </span>
                {/if}
              </button>

              {#if currentUser?.role === "admin" && !isQrMode}
                <button
                  class="w-full flex items-center space-x-3 px-3.5 py-3 rounded-xl font-bold text-xs transition-all cursor-pointer {activeTab === 'devices' ? 'bg-[#e31b23] text-white shadow-md' : 'text-slate-300 hover:bg-white/10 hover:text-white'} {isSidebarExpanded ? '' : 'justify-center px-0'}"
                  on:click={() => switchTab("devices")}
                  title="Device Management"
                >
                  <svg class="w-5 h-5 shrink-0 fill-current" viewBox="0 0 24 24">
                    <path d="M4 6h16v10H4V6zm16 12c1.1 0 2-.9 2-2V6c0-1.1-.9-2-2-2H4c-1.1 0-2 .9-2 2v10c0 1.1.9 2 2 2H0v2h24v-2h-4z"/>
                  </svg>
                  {#if isSidebarExpanded}
                    <span class="truncate">Device Management</span>
                  {/if}
                </button>
              {/if}

              {#if currentUser?.role === "admin"}
                <button
                  class="w-full flex items-center space-x-3 px-3.5 py-3 rounded-xl font-bold text-xs transition-all cursor-pointer {activeTab === 'users' ? 'bg-[#e31b23] text-white shadow-md' : 'text-slate-300 hover:bg-white/10 hover:text-white'} {isSidebarExpanded ? '' : 'justify-center px-0'}"
                  on:click={() => switchTab("users")}
                  title={isQrMode ? "Site Leader & Dynamic Site Control" : "Kiosk Operator Control"}
                >
                  <svg class="w-5 h-5 shrink-0 fill-current" viewBox="0 0 24 24">
                    <path d="M12 12c2.21 0 4-1.79 4-4s-1.79-4-4-4-4 1.79-4 4 1.79 4 4 4zm0 2c-2.67 0-8 1.34-8 4v2h16v-2c0-2.66-5.33-4-8-4z"/>
                  </svg>
                  {#if isSidebarExpanded}
                    <span class="truncate">{isQrMode ? 'Site Leader Control' : 'User Control'}</span>
                  {/if}
                </button>
              {/if}

            </nav>
          </div>

          <!-- CLEAN FOOTER: TARGET TEXT REMOVED -->
          <div class="p-3 theme-border border-t bg-black/10 text-center">
            <span class="h-2 w-2 rounded-full inline-block bg-[#e31b23] animate-pulse" title="System Active"></span>
          </div>

        </div>
      </aside>
    {/if}

    <!-- MAIN CANVAS -->
    <div class="flex-1 flex flex-col h-full min-w-0 max-w-full overflow-hidden relative">
      
      {#if !isDedicatedKioskMode && activeTab !== 'kiosk'}
        <header class="sticky top-0 z-30 w-full h-16 theme-bg-card theme-border border-b flex items-center justify-between px-4 sm:px-6 shrink-0 box-border transition-colors duration-300 theme-shadow">
          <div class="flex items-center space-x-3 min-w-0">
            {#if !isSidebarExpanded}
              <div class="flex items-center space-x-2 shrink-0 transition-all duration-300">
                <div class="h-7 w-7 rounded-lg bg-[#e31b23] flex items-center justify-center font-extrabold text-xs text-white shadow-md">
                  {isQrMode ? 'QR' : 'EK'}
                </div>
                <span class="font-bold text-sm tracking-tight theme-text-primary">
                  {isQrMode ? 'Web QR Hub' : 'Enterprise Kiosk'}
                </span>
              </div>
            {/if}
          </div>

          <div class="flex items-center space-x-3 shrink-0">
            {#if currentUser?.role === 'admin'}
              <button
                on:click={toggleAppMode}
                class="px-3 py-1.5 rounded-xl text-xs font-mono font-bold border transition-all flex items-center space-x-2 shadow-xs cursor-pointer {isQrMode ? 'bg-cyan-50 dark:bg-cyan-950 border-cyan-300 dark:border-cyan-500 text-cyan-900 dark:text-cyan-200' : 'bg-slate-100 dark:bg-slate-800 border-slate-300 dark:border-slate-700 text-slate-800 dark:text-cyan-400 hover:bg-slate-200 dark:hover:bg-slate-700'}"
                title="Toggle App Program Engine (Ctrl + M)"
              >
                <!-- PROFESSIONAL SVG ICONS REPLACING EMOJIS -->
                {#if isQrMode}
                  <svg class="w-4 h-4 fill-current text-cyan-600 dark:text-cyan-400" viewBox="0 0 24 24">
                    <path d="M17 1.01L7 1c-1.1 0-2 .9-2 2v18c0 1.1.9 2 2 2h10c1.1 0 2-.9 2-2V3c0-1.1-.9-1.99-2-1.99zM17 19H7V5h10v14z"/>
                  </svg>
                  <span class="font-bold">Mode: Web QR Hub</span>
                {:else}
                  <svg class="w-4 h-4 fill-current text-cyan-600 dark:text-cyan-400" viewBox="0 0 24 24">
                    <path d="M21 2H3c-1.1 0-2 .9-2 2v12c0 1.1.9 2 2 2h7v2H8v2h8v-2h-2v-2h7c1.1 0 2-.9 2-2V4c0-1.1-.9-2-2-2zm0 14H3V4h18v12z"/>
                  </svg>
                  <span class="font-bold">Mode: Enterprise Kiosk</span>
                {/if}

                <kbd 
                  class="px-2 py-0.5 text-[10px] rounded font-mono font-extrabold border shadow-xs"
                  style={isDarkMode 
                    ? "background-color: #0f172a !important; color: #f8fafc !important; border-color: #334155 !important;" 
                    : "background-color: #ffffff !important; color: #0f172a !important; border-color: #cbd5e1 !important;"}
                >
                  Ctrl+M
                </kbd>
              </button>
            {/if}

            <button
              on:click={toggleTheme}
              class="relative flex items-center px-3 py-1.5 rounded-full theme-border border theme-bg-inner hover:opacity-80 transition-all duration-300 active:scale-95 shadow-inner cursor-pointer"
              title={isDarkMode ? "Switch to Crisp Light Mode" : "Switch to Deep Dark Mode"}
            >
              <div class="flex items-center space-x-2 text-xs font-semibold">
                {#if isDarkMode}
                  <!-- PROFESSIONAL MOON SVG -->
                  <svg class="w-3.5 h-3.5 fill-current text-amber-400" viewBox="0 0 24 24">
                    <path d="M12.3 2a10 10 0 0 0-.19 20 10.04 10.04 0 0 0 9.8-7.73 1 1 0 0 0-1.25-1.21A8 8 0 0 1 10.94 4.34a1 1 0 0 0-1.21-1.25c-.47-.7-.95-.97-1.43-1.09z"/>
                  </svg>
                  <span class="theme-text-secondary text-[11px] hidden md:inline">Dark</span>
                {:else}
                  <!-- PROFESSIONAL SUN SVG -->
                  <svg class="w-3.5 h-3.5 fill-current text-amber-500" viewBox="0 0 24 24">
                    <path d="M12 7c-2.76 0-5 2.24-5 5s2.24 5 5 5 5-2.24 5-5-2.24-5-5-5zM2 13h2c.55 0 1-.45 1-1s-.45-1-1-1H2c-.55 0-1 .45-1 1s.45 1 1 1zm18 0h2c.55 0 1-.45 1-1s-.45-1-1-1h-2c-.55 0-1 .45-1 1s.45 1 1 1zM11 2v2c0 .55.45 1 1 1s1-.45 1-1V2c0-.55-.45-1-1-1s-1 .45-1 1zm0 18v2c0 .55.45 1 1 1s1-.45 1-1v-2c0-.55-.45-1-1-1s-1 .45-1 1zM5.99 4.58a.996.996 0 0 0-1.41 0 .996.996 0 0 0 0 1.41l1.06 1.06c.39.39 1.03.39 1.41 0s.39-1.03 0-1.41L5.99 4.58zm12.37 12.37a.996.996 0 0 0-1.41 0 .996.996 0 0 0 0 1.41l1.06 1.06c.39.39 1.03.39 1.41 0s.39-1.03 0-1.41l-1.06-1.06zm1.06-10.96a.996.996 0 0 0 0-1.41.996.996 0 0 0-1.41 0l-1.06 1.06c-.39.39-.39 1.03 0 1.41s1.03.39 1.41 0l1.06-1.06zM7.05 18.36a.996.996 0 0 0 0-1.41.996.996 0 0 0 0 1.41l1.06 1.06c-.39.39-.39 1.03 0 1.41s1.03.39 1.41 0l1.06-1.06z"/>
                  </svg>
                  <span class="theme-text-primary text-[11px] font-bold hidden md:inline">Light</span>
                {/if}
              </div>
            </button>

            {#if currentUser}
              <div class="flex items-center space-x-2 theme-bg-inner theme-border border px-3 py-1.5 rounded-xl text-xs font-mono shadow-inner">
                <span class="theme-text-secondary truncate">User: <strong class="theme-text-primary">{currentUser.username}</strong></span>
                <span 
                  class="px-2 py-0.5 rounded-md text-[10px] font-extrabold uppercase shrink-0"
                  style={currentUser.role === 'admin' ? 'background-color: #e31b23 !important; color: #ffffff !important;' : 'background-color: #0284c7 !important; color: #ffffff !important;'}
                >
                  {currentUser.role === 'site_leader' ? 'Site Leader' : (currentUser.role === 'user' ? 'Operator' : currentUser.role.replace('_', ' '))}
                </span>
              </div>

              <button
                on:click={handleLogout}
                class="text-xs text-rose-500 hover:text-rose-600 bg-rose-500/10 hover:bg-rose-500/20 border border-rose-500/30 px-3.5 py-1.5 rounded-xl font-extrabold transition-all active:scale-95 shadow-sm flex items-center space-x-1 shrink-0 cursor-pointer"
              >
                <span>Sign Out</span>
                <span class="text-sm">➔</span>
              </button>
            {/if}

            <div class="hidden sm:flex items-center space-x-2 px-2.5 py-1 rounded-full theme-bg-inner theme-border border text-xs font-mono theme-text-secondary shrink-0">
              <span class="h-2 w-2 rounded-full {isOfflineMode ? 'bg-rose-500' : 'bg-emerald-500'} shadow-sm animate-pulse"></span>
              <span>{isOfflineMode ? "Offline" : "Online"}</span>
            </div>
          </div>
        </header>
      {/if}

      <!-- CANVAS BODY -->
      <main class="flex-1 theme-bg-main overflow-y-auto overflow-x-hidden w-full max-w-full box-border transition-colors duration-300 {activeTab === 'kiosk' || isDedicatedKioskMode ? 'p-0' : 'p-4 sm:p-6 lg:p-8'}">
        <div class="w-full h-full min-w-0 max-w-full {activeTab === 'kiosk' || isDedicatedKioskMode ? '' : 'max-w-7xl mx-auto'}">
          {#if activeTab === "surveys" && currentUser?.role === "admin"}
            <div class="w-full h-full min-w-0">
              <Dashboard
                surveys={surveysList.filter(
                  (s) => !s.isDraft && !String(s._id).startsWith("DRAFT-"),
                )}
                responseCount={validResponsesCount}
                onCreateSurvey={handleCreateNewSurvey}
                onDeleteSurvey={handleDeleteSurvey}
                onEditSurvey={handleSelectAndEdit}
                onTestSurvey={handleSelectAndTest}
                onOpenShareModal={handleOpenShareModal}
              />
            </div>
          {:else if activeTab === "builder" && currentUser?.role === "admin"}
            <div class="w-full h-full min-w-0">
              <FormBuilder
                surveyTitle={activeSurvey.title}
                questions={activeSurvey.questions}
                surveys={surveysList}
                {activeSurveyId}
                {isQrMode}
                onSelectSurvey={(id) => (activeSurveyId = id)}
                onCreateNewSurvey={handleCreateNewSurvey}
                onSaveSurvey={persistActiveSurveyState}
              />
            </div>
          {:else if activeTab === "kiosk"}
            <div class="w-full h-full min-w-0 flex items-center justify-center">
              <Kiosk
                surveyTitle={activeSurvey.title}
                questions={activeSurvey.questions}
                surveys={surveysList}
                {activeSurveyId}
                {isQrMode}
                onSelectSurvey={(id) => (activeSurveyId = id)}
                onSubmitResponse={registerResponse}
              />
            </div>
          {:else if activeTab === "answers"}
            <div class="w-full h-full min-w-0">
              <Answers
                bind:responses
                bind:activeSurveyId
                {isQrMode}
                {currentUser}
                surveys={surveysList.filter(
                  (s) => !s.isDraft && !String(s._id).startsWith("DRAFT-"),
                )}
                onRefreshData={refreshDataLedger}
              />
            </div>
          {:else if activeTab === "devices" && currentUser?.role === "admin" && !isQrMode}
            <div class="w-full h-full min-w-0">
              <DeviceManagement {currentUser} />
            </div>
          {:else if activeTab === "users" && currentUser?.role === "admin"}
            <div class="w-full h-full min-w-0">
              <UserManagement {currentUser} {isQrMode} />
            </div>
          {/if}
        </div>
      </main>

    </div>
  </div>

  <!-- ROOT-LEVEL SHARE HUB MODAL OVERLAY -->
  {#if showShareModal && activeShareSurvey}
    {@const dynamicKioskUrl = getKioskLink(activeShareSurvey, activeShareSurvey.appMode === 'qr')}
    <div class="fixed inset-0 z-[9999] w-screen h-screen bg-slate-900/30 dark:bg-slate-950/70 backdrop-blur-md flex items-center justify-center p-4">
      <div 
        in:scale={{ duration: 250, start: 0.95 }}
        class="bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-800 w-full max-w-sm rounded-3xl p-6 sm:p-8 text-center space-y-6 shadow-2xl relative overflow-hidden"
      >
        <button on:click={handleCloseShareModal} class="absolute top-4 right-4 text-slate-400 hover:text-slate-700 dark:hover:text-white bg-slate-100 dark:bg-slate-950 border border-slate-200 dark:border-slate-800/80 h-8 w-8 rounded-full flex items-center justify-center text-xs transition-all active:scale-95 cursor-pointer">
          ✕
        </button>

        <div class="space-y-1">
          <span class="text-[10px] font-bold text-[#e31b23] dark:text-rose-400 tracking-widest uppercase block font-mono">
            {activeShareSurvey.appMode === 'qr' ? 'Public Scan QR Code' : 'Enterprise Terminal Link / QR'}
          </span>
          <h3 class="text-base sm:text-lg font-extrabold text-[#1a2b6c] dark:text-white truncate max-w-[280px] mx-auto">{activeShareSurvey.title}</h3>
        </div>

        <div class="bg-white p-4 rounded-2xl inline-block shadow-lg mx-auto border-4 border-slate-100 dark:border-slate-800">
          <img 
            src="https://api.qrserver.com/v1/create-qr-code/?size=180x180&data={encodeURIComponent(dynamicKioskUrl)}&color=1a2b6c" 
            alt="Survey QR Code Link" 
            class="h-44 w-44 block"
          />
        </div>

        <p class="text-xs text-slate-500 dark:text-slate-400 max-w-xs mx-auto px-2 leading-relaxed">
          {activeShareSurvey.appMode === 'qr' 
            ? 'Scan this QR code to load and complete this survey directly on any phone or browser.' 
            : 'Scan or open on a kiosk tablet. Requires Device PIN code before launching.'}
        </p>

        {#if copyBannerMessage}
          <div class="bg-emerald-50 dark:bg-emerald-950/80 border border-emerald-300 dark:border-emerald-800 text-emerald-800 dark:text-emerald-300 text-xs font-bold py-2 px-3 rounded-xl animate-fade font-mono">
            {copyBannerMessage}
          </div>
        {/if}

        <div class="pt-2 border-t border-slate-100 dark:border-slate-800/60">
          <button 
            type="button"
            on:click={() => copyKioskLink(activeShareSurvey, activeShareSurvey.appMode === 'qr')} 
            class="w-full bg-[#1a2b6c] hover:bg-[#e31b23] font-bold py-3.5 px-4 text-xs rounded-xl transition-all shadow-md active:scale-95 flex items-center justify-center space-x-2 cursor-pointer"
            style="color: #ffffff !important; font-weight: 700 !important; background-color: #1a2b6c !important;"
          >
            <svg class="w-4 h-4 fill-current text-white" viewBox="0 0 24 24" style="fill: #ffffff !important;">
              <path d="M3.9 12c0-1.71 1.39-3.1 3.1-3.1h4V7H7c-2.76 0-5 2.24-5 5s2.24 5 5 5h4v-1.9H7c-1.71 0-3.1-1.39-3.1-3.1zM8 13h8v-2H8v2zm9-6h-4v1.9h4c1.71 0 3.1 1.39 3.1 3.1s-1.39 3.1-3.1 3.1h-4V17h4c2.76 0 5-2.24 5-5s-2.24-5-5-5z"/>
            </svg>
            <span style="color: #ffffff !important; font-weight: 700 !important;">Copy Share Link</span>
          </button>
        </div>
      </div>
    </div>
  {/if}
{/if}