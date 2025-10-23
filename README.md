<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Hoops & Hustle Tracker</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script type="module">
        // Firebase Imports (MUST be used for persistence)
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, getDoc, addDoc, setDoc, updateDoc, deleteDoc, onSnapshot, collection, query, where, orderBy, getDocs, arrayUnion } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // --- GLOBAL VARIABLES (Provided by the environment) ---
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : { /* mock config for local testing */ };
        // FIX: Corrected variable access from 'initialAuthToken' to '__initial_auth_token'
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        let db;
        let auth;
        let userId = 'loading';
        const SESSION_COLLECTION_BASE = `artifacts/${appId}/public/data/basketball_sessions`;
        let SESSION_COLLECTION; // Set after userId is known
        let PLAYER_PROFILE_COLLECTION; // Private collection for player balances
        let playerProfiles = {}; // Cached player profile data { name: { name, balance } }
        let allSessions = []; // Global cache for sessions to use in export

        // Utility to format date YYYY-MM-DD
        const getFormattedDate = (date) => new Date(date).toISOString().split('T')[0];
        
        // Utility to format currency (handles balances stored in CENTS)
        const formatCurrency = (amountInCents) => {
            const dollars = amountInCents / 100;
            return new Intl.NumberFormat('en-US', {
                style: 'currency',
                currency: 'USD',
                minimumFractionDigits: 2,
            }).format(dollars);
        };
        
        // Utility to format balance as raw dollars for CSV
        const formatDollars = (amountInCents) => (amountInCents / 100).toFixed(2);


        // Utility to handle error display (instead of alert)
        const showMessage = (text, isError = false) => {
            const container = document.getElementById('message-container');
            container.textContent = text;
            container.className = `p-4 mt-4 rounded-lg text-sm transition-opacity ${isError ? 'bg-red-100 text-red-800' : 'bg-green-100 text-green-800'} opacity-100`;
            setTimeout(() => {
                container.classList.remove('opacity-100');
                container.classList.add('opacity-0');
            }, 3000);
        };

        // --- FIREBASE INITIALIZATION & AUTHENTICATION ---
        const initFirebase = async () => {
            try {
                // setLogLevel('Debug'); // Uncomment for debugging
                const app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);

                // Authenticate user
                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                } else {
                    await signInAnonymously(auth);
                }

                onAuthStateChanged(auth, (user) => {
                    if (user) {
                        userId = user.uid;
                    } else {
                        userId = crypto.randomUUID(); // Fallback to a random ID
                    }
                    
                    SESSION_COLLECTION = SESSION_COLLECTION_BASE; // Public for all users to see sessions
                    // Private collection for player balances, scoped to the host (userId)
                    PLAYER_PROFILE_COLLECTION = `artifacts/${appId}/users/${userId}/player_profiles`;

                    document.getElementById('user-id-display').textContent = `Host ID: ${userId}`;
                    
                    loadPlayerProfiles(); // Load balances first
                    loadSessions();      // Then load session data
                });

            } catch (error) {
                console.error("Firebase Initialization Error:", error);
                showMessage(`Error setting up Firebase: ${error.message}`, true);
                document.getElementById('user-id-display').textContent = 'Error';
            }
        };

        // --- PLAYER PROFILE BALANCE LOGIC ---

        const loadPlayerProfiles = () => {
            if (!db || !PLAYER_PROFILE_COLLECTION) return;

            // Listen for real-time updates on player balances
            onSnapshot(collection(db, PLAYER_PROFILE_COLLECTION), (snapshot) => {
                playerProfiles = {};
                snapshot.forEach(doc => {
                    // doc.id is the player's name
                    playerProfiles[doc.id] = doc.data();
                });
                console.log(`Loaded ${Object.keys(playerProfiles).length} player profiles.`);

                // Re-render sessions to update balance displays immediately
                const currentSessionData = document.getElementById('sessions-container').dataset.sessions;
                if (currentSessionData) {
                    renderSessions(JSON.parse(currentSessionData));
                }
            }, (error) => {
                console.error("Error fetching player profiles:", error);
            });
        };

        const createOrUpdatePlayerBalance = async (playerName, balanceChange) => {
            if (!db) return;
            const playerRef = doc(db, PLAYER_PROFILE_COLLECTION, playerName);
            const snap = await getDoc(playerRef);
            let newBalance = balanceChange;

            if (snap.exists()) {
                // Balance is stored in cents (number), default to 0 if field is missing
                const currentBalance = snap.data().balance || 0; 
                newBalance = currentBalance + balanceChange;
                await updateDoc(playerRef, { balance: newBalance });
            } else {
                // New player profile, set initial balance
                await setDoc(playerRef, { name: playerName, balance: balanceChange });
            }
        };


        // --- GEMINI API INTEGRATION ---
        const GEMINI_API_URL = "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=";
        const API_KEY = ""; // Placeholder for Canvas environment

        const geminiApiCall = async (userQuery, systemPrompt, retries = 3) => {
            const apiUrl = `${GEMINI_API_URL}${API_KEY}`;
            const payload = {
                contents: [{ parts: [{ text: userQuery }] }],
                systemInstruction: { parts: [{ text: systemPrompt }] },
            };

            for (let i = 0; i < retries; i++) {
                try {
                    const response = await fetch(apiUrl, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify(payload)
                    });

                    if (!response.ok) {
                        if (response.status === 429 && i < retries - 1) {
                            const delay = Math.pow(2, i) * 1000;
                            console.warn(`Rate limit hit. Retrying in ${delay / 1000}s...`);
                            await new Promise(resolve => setTimeout(resolve, delay));
                            continue;
                        }
                        throw new Error(`API Error: ${response.statusText}`);
                    }

                    const result = await response.json();
                    const text = result.candidates?.[0]?.content?.parts?.[0]?.text;
                    return text || "Error generating content: Model returned empty response.";

                } catch (error) {
                    console.error("Gemini API call failed:", error);
                    if (i === retries - 1) throw new Error(`Final API Call Failed: ${error.message}`);
                }
            }
            return "Failed to get a response from the AI service after multiple retries.";
        };


        // LLM Feature 1: Generate Session Summary/Hype Message
        const generateSessionSummary = async (session) => {
            if (!db) return;
            
            // Show loading state
            const summaryOutput = document.getElementById(`summary-output-${session.id}`);
            summaryOutput.innerHTML = '<p class="text-pink-600 font-medium">Generating hoops summary... 🏀</p>';

            try {
                const paidPlayers = session.playerData.filter(p => p.paid).map(p => p.name).join(', ');
                const unpaidPlayers = session.playerData.filter(p => !p.paid).map(p => p.name).join(', ');
                const totalPlayers = session.playerData.length;

                const userQuery = `
                    Session Date: ${session.date}.
                    Total Players: ${totalPlayers}.
                    Players who have paid: ${paidPlayers || 'None yet.'}.
                    Players who still need to pay: ${unpaidPlayers || 'Everyone is paid!'}.
                    The cost per player for this session is ${formatCurrency(session.costPerPlayer || 0)}.
                `;

                const systemPrompt = `
                    You are a charismatic basketball league manager. Your task is to generate a short, fun, and motivating session summary or roster update announcement based on the provided player data.
                    The tone should be energetic and friendly.
                    - Congratulate those who have paid.
                    - Gently remind those who haven't paid (mention the need to cover gym costs).
                    - Keep the response concise, engaging, and in a single paragraph. Do not use bullet points or lists.
                    - If everyone is paid, focus on hype and teamwork.
                `;

                const summary = await geminiApiCall(userQuery, systemPrompt);

                // Display summary
                summaryOutput.innerHTML = `<div class="mt-3 p-4 bg-gray-50 border border-gray-200 rounded-lg text-gray-700 whitespace-pre-wrap">${summary}</div>`;

            } catch (error) {
                summaryOutput.innerHTML = `<p class="text-red-500 text-sm">Error: ${error.message}</p>`;
            }
        };

        // LLM Feature 2: Draft Payment Reminder Message
        const draftPaymentReminder = async (session) => {
            if (!db) return;

            const reminderOutput = document.getElementById(`reminder-output-${session.id}`);
            reminderOutput.innerHTML = '<p class="text-purple-600 font-medium">Drafting reminder message... ✍️</p>';

            try {
                const unpaidPlayers = session.playerData.filter(p => !p.paid).map(p => p.name);
                const paidPlayersCount = session.playerData.filter(p => p.paid).length;
                const cost = formatCurrency(session.costPerPlayer || 0);
                
                if (unpaidPlayers.length === 0) {
                    reminderOutput.innerHTML = `<p class="mt-3 p-4 bg-green-50 border border-green-200 rounded-lg text-green-700">All set! Everyone for the ${session.date} session is paid up. No reminder needed!</p>`;
                    return;
                }

                const userQuery = `
                    Session Date: ${session.date}.
                    Cost Per Player: ${cost}.
                    Players who still need to pay (${unpaidPlayers.length} people): ${unpaidPlayers.join(', ')}.
                    ${paidPlayersCount} players have already paid.
                    Goal: Draft a short, actionable text reminder to send to the group chat.
                `;

                const systemPrompt = `
                    You are a friendly and efficient host of a weekly basketball game. Draft a casual, brief, and polite text message (only the message body, no salutation or sign-off) to be shared with the group.
                    The message must:
                    1. Clearly state the session date and the cost (e.g., "$10").
                    2. List the names of the people who still owe money for the session.
                    3. Include a very polite call to action to send the payment soon.
                    4. Keep it conversational and brief.
                `;

                const reminder = await geminiApiCall(userQuery, systemPrompt);

                // Display summary
                reminderOutput.innerHTML = `
                    <div class="mt-3">
                        <p class="text-sm font-semibold text-gray-700 mb-2">Copy & Share Reminder:</p>
                        <div class="bg-indigo-50 p-4 border border-indigo-200 rounded-lg text-gray-800 whitespace-pre-wrap relative">
                            ${reminder}
                        </div>
                    </div>
                `;

            } catch (error) {
                reminderOutput.innerHTML = `<p class="text-red-500 text-sm">Error drafting reminder: ${error.message}</p>`;
            }
        };
        
        // LLM Feature 3: Generate Session Prep Checklist
        const generatePrepChecklist = async (session) => {
            if (!db) return;

            const checklistOutput = document.getElementById(`checklist-output-${session.id}`);
            checklistOutput.innerHTML = '<p class="text-green-600 font-medium">Generating prep checklist... 📋</p>';

            try {
                const userQuery = `
                    Generate a preparation checklist for the basketball session scheduled for ${session.date}.
                    The cost per player is ${formatCurrency(session.costPerPlayer || 0)}.
                `;

                const systemPrompt = `
                    You are an organized and helpful session coordinator. Generate a short, actionable, bulleted checklist (5-7 items) for the host to prepare for their basketball session on the provided date.
                    The list should cover logistics, player communication, and equipment checks.
                    Include a brief, motivational introduction before the list.
                    The output must be formatted using markdown bullet points (*).
                `;

                const checklist = await geminiApiCall(userQuery, systemPrompt);

                // Display checklist
                checklistOutput.innerHTML = `
                    <div class="mt-3">
                        <div class="bg-gray-50 p-4 border border-gray-200 rounded-lg text-gray-700 whitespace-pre-wrap relative">
                            ${checklist}
                        </div>
                    </div>
                `;

            } catch (error) {
                checklistOutput.innerHTML = `<p class="text-red-500 text-sm">Error generating checklist: ${error.message}</p>`;
            }
        };

        // --- SEEDING LOGIC ---
        const seedInitialData = async () => {
            if (!db || !SESSION_COLLECTION || userId === 'loading') return;

            // Double-check to prevent re-seeding if called multiple times before data is ready
            const snap = await getDocs(collection(db, SESSION_COLLECTION));
            if (!snap.empty) {
                return; // Data already exists
            }

            const today = getFormattedDate(new Date());
            const hostName = "Host (You)";
            const costPerPlayerCents = 1000; // $10.00

            try {
                // 1. Create Host Profile (Balance 0)
                await createOrUpdatePlayerBalance(hostName, 0); 

                // 2. Add sample session with default players
                const newSession = {
                    date: today,
                    hostId: userId,
                    costPerPlayer: costPerPlayerCents, 
                    playerData: [
                        { name: hostName, paid: true },
                        { name: 'Alex', paid: false }, // Owes $10.00
                        { name: 'Ben', paid: true },   // Paid
                        { name: 'Chris', paid: false },// Owes $10.00
                    ],
                    createdAt: Date.now()
                };

                await addDoc(collection(db, SESSION_COLLECTION), newSession);

                // 3. Update balances for the sample players based on UNPAID status
                // Alex is UNPAID: owes $10.00
                await createOrUpdatePlayerBalance('Alex', costPerPlayerCents);
                
                // Chris is UNPAID: owes $10.00
                await createOrUpdatePlayerBalance('Chris', costPerPlayerCents);
                
                // Ben is PAID, balance change is 0 (he would have been added as UNPAID and then paid, resulting in net zero).
                await createOrUpdatePlayerBalance('Ben', 0);

                showMessage(`Tracker is ready! One sample session for ${today} has been created for you.`, false);

            } catch (error) {
                console.error("Error seeding initial data:", error);
            }
        };


        // --- DATA OPERATIONS ---

        const loadSessions = () => {
            if (!db || !SESSION_COLLECTION) return;

            // Query to listen for all public sessions
            const sessionsQuery = collection(db, SESSION_COLLECTION);

            onSnapshot(sessionsQuery, (snapshot) => {
                allSessions = [];
                snapshot.forEach(doc => {
                    const data = doc.data();
                    if (!data.playerData) data.playerData = [];
                    allSessions.push({ id: doc.id, ...data });
                });
                // Sort by createdAt descending (latest session first)
                allSessions.sort((a, b) => (b.createdAt || 0) - (a.createdAt || 0));
                
                // Store sessions data on the container for access when profiles update
                document.getElementById('sessions-container').dataset.sessions = JSON.stringify(allSessions);

                renderSessions(allSessions);
                
                // --- SEEDING LOGIC: Check if data is empty and seed if necessary ---
                if (allSessions.length === 0) {
                    seedInitialData();
                }
                // --- END SEEDING LOGIC ---

            }, (error) => {
                console.error("Error fetching sessions:", error);
                showMessage("Could not load sessions in real-time.", true);
            });
        };

        const addNewSession = async () => {
            if (!db || userId === 'loading') {
                 showMessage("App not ready. Please wait for initialization.", true);
                 return;
            }

            const costInput = document.getElementById('session-cost');
            const costPerPlayerUSD = parseFloat(costInput.value);

            if (isNaN(costPerPlayerUSD) || costPerPlayerUSD <= 0) {
                showMessage("Please enter a valid cost per player (e.g., 10.00).", true);
                return;
            }
            // Store cost in cents to avoid floating point issues
            const costPerPlayerCents = Math.round(costPerPlayerUSD * 100); 

            const today = getFormattedDate(new Date());
            const hostName = "Host (You)";

            try {
                // Ensure host profile exists (Host is always assumed paid initially, so balance is 0 for this session)
                await createOrUpdatePlayerBalance(hostName, 0); 

                const newSession = {
                    date: today,
                    hostId: userId,
                    costPerPlayer: costPerPlayerCents, // Store cost in cents
                    playerData: [
                        { name: hostName, paid: true } // Host is assumed paid initially
                    ],
                    createdAt: Date.now()
                };

                await addDoc(collection(db, SESSION_COLLECTION), newSession);
                showMessage(`New session added for ${today}! Cost: ${formatCurrency(costPerPlayerCents)}.`);

            } catch (error) {
                console.error("Error adding new session:", error);
                showMessage(`Failed to add new session: ${error.message}`, true);
            }
        };

        const updatePlayerStatus = async (sessionId, playerName, newStatus) => {
            if (!db) return;

            try {
                const sessionRef = doc(db, SESSION_COLLECTION, sessionId);
                const sessionSnap = await getDoc(sessionRef);

                if (sessionSnap.exists()) {
                    const sessionData = sessionSnap.data();
                    const cost = sessionData.costPerPlayer || 0; // Cost in cents

                    let updatedPlayers = sessionData.playerData;
                    let playerEntry = updatedPlayers.find(p => p.name === playerName);

                    if (!playerEntry) return;

                    let balanceChange = 0;
                    const statusChanged = playerEntry.paid !== newStatus;

                    if (statusChanged) {
                        // Calculate change based on current state and new state
                        if (newStatus === true) {
                            // UNPAID (debt) -> PAID (clearing debt/paying host)
                            // Player's balance decreases by the cost (i.e., they paid)
                            balanceChange = -cost; 
                        } else { 
                            // PAID -> UNPAID (creating debt/requesting refund)
                            // Player's balance increases by the cost (i.e., cost is now owed again)
                            balanceChange = cost; 
                        }

                        // 1. Update Player Profile Balance
                        await createOrUpdatePlayerBalance(playerName, balanceChange);

                        // 2. Update Session Data
                        updatedPlayers = updatedPlayers.map(player => {
                            if (player.name === playerName) {
                                return { ...player, paid: newStatus };
                            }
                            return player;
                        });

                        await updateDoc(sessionRef, {
                            playerData: updatedPlayers
                        });
                        showMessage(`Status and balance updated for ${playerName}.`);
                    }

                }
            } catch (error) {
                console.error("Error updating player status and balance:", error);
                showMessage("Failed to update payment status and balance.", true);
            }
        };

        const addPlayerToSession = async (sessionId, playerName) => {
            if (!db) return;

            const name = playerName.trim();
            if (!name) {
                showMessage("Player name cannot be empty.", true);
                return;
            }

            try {
                const sessionRef = doc(db, SESSION_COLLECTION, sessionId);
                const sessionSnap = await getDoc(sessionRef);

                if (sessionSnap.exists()) {
                    const sessionData = sessionSnap.data();
                    const playerData = sessionData.playerData || [];

                    // Check for duplicates
                    if (playerData.some(p => p.name.toLowerCase() === name.toLowerCase())) {
                        showMessage(`${name} is already in this session.`, true);
                        return;
                    }

                    const cost = sessionData.costPerPlayer || 0;
                    
                    // 1. Update Player Balance (Create profile and add session cost as initial debt)
                    // Player is UNPAID by default, so their debt increases by the cost
                    await createOrUpdatePlayerBalance(name, cost);

                    // 2. Add to session (UNPAID by default)
                    const newPlayer = { name: name, paid: false };
                    const newPlayerData = [...playerData, newPlayer];

                    await updateDoc(sessionRef, {
                        playerData: newPlayerData
                    });
                    
                    showMessage(`${name} added! Cost of ${formatCurrency(cost)} added to their balance.`);
                }
            } catch (error) {
                console.error("Error adding player:", error);
                showMessage(`Failed to add player: ${error.message}`, true);
            }
        };

        // --- EXPORT LOGIC ---

        const exportToCsv = () => {
            if (allSessions.length === 0) {
                showMessage("No session data to export.", true);
                return;
            }

            // 1. Collect all unique player names across all sessions
            const uniquePlayers = new Set();
            allSessions.forEach(session => {
                session.playerData.forEach(p => uniquePlayers.add(p.name));
            });
            const playerNames = Array.from(uniquePlayers).sort();

            let csvContent = "";
            const today = getFormattedDate(new Date());

            // --- SECTION A: SESSION HISTORY BREAKDOWN ---
            csvContent += `SECTION: SESSION HISTORY BREAKDOWN\n`;
            
            // Generate dynamic header for session data
            const sessionHeader = ["Date", "Session Cost (USD)", "Total Players", "Paid Count", ...playerNames];
            csvContent += sessionHeader.join(",") + "\n";

            // Generate rows for session data
            allSessions.forEach(session => {
                const row = [];
                row.push(session.date);
                row.push(formatDollars(session.costPerPlayer || 0));
                row.push(session.playerData.length);
                row.push(session.playerData.filter(p => p.paid).length);

                // Create a map for quick player status lookup in this session
                const sessionPlayerMap = new Map(session.playerData.map(p => [p.name, p.paid ? "PAID" : "UNPAID"]));

                // Add status for every unique player
                playerNames.forEach(name => {
                    const status = sessionPlayerMap.get(name) || "NOT ATTENDED";
                    row.push(status);
                });

                csvContent += row.join(",") + "\n";
            });

            // --- SECTION B: PLAYER RUNNING BALANCES ---
            csvContent += "\n\nSECTION: PLAYER RUNNING BALANCES (Balance Carried Over)\n";
            
            // Header for balance data
            const balanceHeader = ["Player Name", "Balance (USD)"];
            csvContent += balanceHeader.join(",") + "\n";

            // Generate rows for balance data
            const sortedPlayerNames = Object.keys(playerProfiles).sort();
            sortedPlayerNames.forEach(name => {
                const balanceInCents = playerProfiles[name]?.balance || 0;
                const balanceUSD = formatDollars(balanceInCents);
                csvContent += `${name},${balanceUSD}\n`;
            });

            // Trigger download
            const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
            const link = document.createElement("a");
            if (link.download !== undefined) { 
                const url = URL.createObjectURL(blob);
                link.setAttribute("href", url);
                link.setAttribute("download", `basketball_tracker_export_${today}.csv`);
                link.style.visibility = 'hidden';
                document.body.appendChild(link);
                link.click();
                document.body.removeChild(link);
                showMessage("Data successfully exported to CSV!", false);
            }
        };


        // --- UI RENDERING ---

        const renderSessions = (sessions) => {
            const container = document.getElementById('sessions-container');
            container.innerHTML = ''; // Clear existing content

            if (sessions.length === 0) {
                container.innerHTML = '<p class="text-center text-gray-500 mt-8">No sessions recorded yet. Click "Start Session" to begin!</p>';
                return;
            }

            sessions.forEach(session => {
                const totalPlayers = session.playerData.length;
                const paidPlayers = session.playerData.filter(p => p.paid).length;

                const sessionCard = document.createElement('div');
                const today = getFormattedDate(new Date());
                const isCurrentSession = session.date === today; 
                
                sessionCard.className = `bg-white p-5 rounded-xl shadow-lg border-2 mb-6 ${isCurrentSession ? 'border-indigo-500 ring-4 ring-indigo-200' : 'border-gray-200'}`;
                sessionCard.id = `session-${session.id}`;

                const playersHtml = session.playerData.map(player => {
                    const balance = playerProfiles[player.name]?.balance || 0;
                    const balanceClass = balance > 0 ? 'text-red-600 font-semibold' : (balance < 0 ? 'text-green-600 font-semibold' : 'text-gray-500');
                    // Balance is positive (Owes), Balance is negative (Credit/Paid Extra), Balance is zero (Settled)
                    const balanceText = balance > 0 ? `Owes: ${formatCurrency(balance)}` : (balance < 0 ? `Credit: ${formatCurrency(Math.abs(balance))}` : 'Settled');

                    return `
                        <div data-name="${player.name}" class="player-item flex justify-between items-center p-3 rounded-lg cursor-pointer transition duration-150 ease-in-out ${player.paid ? 'bg-green-50 hover:bg-green-100' : 'bg-gray-50 hover:bg-gray-100'} shadow-sm">
                            <div class="flex flex-col">
                                <span class="font-medium text-gray-700">${player.name}</span>
                                <span class="text-xs ${balanceClass}">${balanceText}</span>
                            </div>
                            <span class="paid-status-badge px-3 py-1 rounded-full text-xs font-semibold ${player.paid ? 'bg-green-500 text-white' : 'bg-gray-400 text-gray-800'}">
                                ${player.paid ? 'PAID' : 'UNPAID'}
                            </span>
                        </div>
                    `;
                }).join('');


                sessionCard.innerHTML = `
                    <div class="flex justify-between items-start mb-4">
                        <h2 class="text-2xl font-extrabold text-gray-800 flex flex-col items-start">
                            <span class="text-indigo-600 text-sm font-medium">Session Cost: ${formatCurrency(session.costPerPlayer || 0)}</span>
                            <span class="flex items-center mt-1">
                                <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6 text-indigo-600 mr-2" viewBox="0 0 20 20" fill="currentColor">
                                    <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm1-12a1 1 0 10-2 0v4a1 1 0 00.293.707l2.828 2.829a1 1 0 101.415-1.415L11 9.586V6z" clip-rule="evenodd" />
                                </svg>
                                ${session.date}
                            </span>
                        </h2>
                        <div class="text-right">
                             <p class="text-xl font-bold ${paidPlayers === totalPlayers ? 'text-green-600' : 'text-red-500'}">
                                ${paidPlayers}/${totalPlayers} Paid
                            </p>
                            <p class="text-xs text-gray-500">Host: ${session.hostId.substring(0, 8)}...</p>
                        </div>
                    </div>

                    <div id="players-list-${session.id}" class="space-y-3 mb-6">
                        ${playersHtml}
                    </div>

                    <div class="flex space-x-2">
                        <input id="new-player-input-${session.id}" type="text" placeholder="New Player Name" class="flex-grow p-2 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500">
                        <button id="add-player-btn-${session.id}" class="bg-indigo-600 text-white px-4 py-2 rounded-lg font-semibold hover:bg-indigo-700 transition duration-150 shadow-md">
                            Add Player
                        </button>
                    </div>

                    <!-- Gemini AI Features -->
                    <div class="mt-4 pt-4 border-t border-gray-100 space-y-3">
                        <button id="generate-summary-btn-${session.id}" class="w-full bg-pink-500 text-white px-4 py-2 rounded-lg font-bold hover:bg-pink-600 transition duration-150 shadow-md transform hover:scale-[1.01]">
                            ✨ Generate Session Hype Summary
                        </button>
                        <div id="summary-output-${session.id}" class="mt-3">
                            <!-- AI Summary will appear here -->
                        </div>
                        
                        <button id="draft-reminder-btn-${session.id}" class="w-full bg-purple-500 text-white px-4 py-2 rounded-lg font-bold hover:bg-purple-600 transition duration-150 shadow-md transform hover:scale-[1.01]">
                            ✨ Draft Payment Reminder
                        </button>
                         <div id="reminder-output-${session.id}" class="mt-3">
                            <!-- AI Reminder will appear here -->
                        </div>
                        
                        <button id="generate-checklist-btn-${session.id}" class="w-full bg-green-500 text-white px-4 py-2 rounded-lg font-bold hover:bg-green-600 transition duration-150 shadow-md transform hover:scale-[1.01]">
                            ✨ Generate Prep Checklist
                        </button>
                         <div id="checklist-output-${session.id}" class="mt-3">
                            <!-- AI Checklist will appear here -->
                        </div>
                    </div>
                    <!-- End Gemini AI Features -->
                `;

                container.appendChild(sessionCard);

                // Add event listeners for payment toggling
                session.playerData.forEach(player => {
                    const playerElement = sessionCard.querySelector(`.player-item[data-name="${player.name}"]`);
                    if (playerElement) {
                        playerElement.addEventListener('click', () => {
                            updatePlayerStatus(session.id, player.name, !player.paid);
                        });
                    }
                });

                // Add event listener for adding new player
                document.getElementById(`add-player-btn-${session.id}`).addEventListener('click', () => {
                    const input = document.getElementById(`new-player-input-${session.id}`);
                    addPlayerToSession(session.id, input.value);
                    input.value = ''; // Clear input after submission
                });
                
                // Add event listener for generating summary (Feature 1)
                document.getElementById(`generate-summary-btn-${session.id}`).addEventListener('click', () => {
                    generateSessionSummary(session);
                });
                
                // Add event listener for drafting reminder (Feature 2)
                document.getElementById(`draft-reminder-btn-${session.id}`).addEventListener('click', () => {
                    draftPaymentReminder(session);
                });
                
                // Add event listener for generating checklist (Feature 3)
                document.getElementById(`generate-checklist-btn-${session.id}`).addEventListener('click', () => {
                    generatePrepChecklist(session);
                });

            });
        };
        
        // --- EXPOSE FUNCTIONS TO GLOBAL WINDOW ---
        window.addNewSession = addNewSession;
        window.exportToCsv = exportToCsv;
        window.generateSessionSummary = generateSessionSummary;
        window.draftPaymentReminder = draftPaymentReminder;
        window.generatePrepChecklist = generatePrepChecklist;


        // --- STARTUP ---
        window.onload = initFirebase;

    </script>
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f4f7f9;
        }
        .container {
            max-width: 640px;
        }
        .whitespace-pre-wrap {
            white-space: pre-wrap; /* Ensures line breaks in the LLM response are respected */
        }
    </style>
</head>
<body>
    <div class="container mx-auto p-4 sm:p-6">
        <!-- Header -->
        <header class="text-center mb-8">
            <h1 class="text-4xl font-black text-gray-900">Hoops & Hustle Tracker</h1>
            <p id="user-id-display" class="text-sm text-gray-500 mt-1">Authenticating...</p>
        </header>

        <!-- Control Panel -->
        <div class="bg-white p-5 rounded-xl shadow-xl mb-8">
            <h2 class="text-lg font-semibold text-gray-800 mb-4">Start New Session & Export</h2>
            <div class="flex flex-col space-y-4">
                <div class="flex items-center space-x-3">
                    <div class="flex-grow">
                        <label for="session-cost" class="text-sm font-medium text-gray-600">Cost Per Player (USD)</label>
                        <input id="session-cost" type="number" step="0.01" value="10.00" class="w-full p-2 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500 mt-1" placeholder="e.g., 10.00">
                    </div>
                    <button onclick="addNewSession()" class="bg-indigo-500 text-white px-6 py-2 rounded-full font-bold shadow-md hover:bg-indigo-600 transition duration-300 transform hover:scale-105 self-end">
                        Start Session
                    </button>
                </div>
                <!-- NEW EXPORT BUTTON -->
                <button onclick="exportToCsv()" class="bg-gray-700 text-white px-6 py-2 rounded-lg font-semibold shadow-md hover:bg-gray-800 transition duration-300">
                    ⬇️ Export Data to CSV
                </button>
            </div>
        </div>

        <!-- Real-time Message Container -->
        <div id="message-container" class="opacity-0 transition-opacity duration-500"></div>

        <!-- Sessions Container -->
        <div id="sessions-container" data-sessions="[]">
            <div class="text-center mt-8 text-gray-500">Loading sessions...</div>
        </div>

    </div>

</body>
</html>
