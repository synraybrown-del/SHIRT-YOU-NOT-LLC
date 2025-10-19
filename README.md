# SHIRT-YOU-NOT-LLC<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Shirt You Not LLC - Design Build & Renovations</title>
    <!-- Load Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Set up Inter font and Custom Colors -->
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    colors: {
                        'dark-brown': '#4D392E', // Deep Espresso/Dark Brown Background
                        'orange-pearl': '#FFB347', // Vibrant Gold/Orange Accent
                        'light-text': '#F5F5F5',
                    },
                    fontFamily: {
                        sans: ['Inter', 'sans-serif'],
                    },
                }
            }
        }
    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@100..900&display=swap');
        
        /* Custom scrollbar for better look on dark theme */
        ::-webkit-scrollbar { width: 8px; }
        ::-webkit-scrollbar-track { background: #33261F; }
        ::-webkit-scrollbar-thumb { background: #FFB347; border-radius: 4px; }
        ::-webkit-scrollbar-thumb:hover { background: #E69D34; }
        
        /* Custom text shadow for the "Orange Pearl" logo effect */
        .text-shadow-orange {
            text-shadow: 0 0 5px rgba(255, 179, 71, 0.8), 0 0 10px rgba(255, 179, 71, 0.6);
        }
    </style>
</head>
<body class="bg-dark-brown text-light-text font-sans antialiased min-h-screen">

    <!-- Firebase Setup: NOTE: The real-time testimonial feature depends on external environment variables (appId, firebaseConfig, auth token) which are available in this current platform but will need to be configured manually if deploying to a standard static site host like GitHub Pages. The code below is set up to handle this gracefully. -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, addDoc, onSnapshot, collection, query, serverTimestamp, setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // --- GLOBAL SETUP ---
        // These variables require configuration outside of this file if running on GitHub Pages
        const appId = 'SYN-LLC-Website'; // Placeholder ID
        const firebaseConfig = {}; // **REPLACE with your actual Firebase project config if deploying**

        let db, auth, userId, isFirebaseReady = false;

        const initFirebase = async () => {
             // Check if Firebase config is empty (typical for GitHub Pages deployment)
            if (Object.keys(firebaseConfig).length === 0) {
                 console.warn("Firebase config is missing. Testimonials are disabled. Please configure Firebase to enable real-time reviews.");
                 document.getElementById('testimonials-list').innerHTML = `<p class="text-yellow-400 p-4 text-center">Testimonials are currently disabled. Please contact the site administrator to configure Firebase for real-time reviews.</p>`;
                 document.getElementById('testimonial-form-container').style.display = 'none';
                 return;
            }

            try {
                const app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);
                setLogLevel('Debug');
                isFirebaseReady = true;

                const __initial_auth_token = typeof window.__initial_auth_token !== 'undefined' ? window.__initial_auth_token : undefined;
                
                if (__initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    await signInAnonymously(auth);
                }
                
                onAuthStateChanged(auth, (user) => {
                    userId = user ? user.uid : 'anonymous-' + crypto.randomUUID();
                    console.log("Firebase Auth State Changed. User ID:", userId);
                    setupTestimonialListener();
                });
            } catch (error) {
                console.error("Firebase Initialization Error:", error);
                document.getElementById('testimonials-list').innerHTML = `<p class="text-red-400 p-4 text-center">Error loading testimonials. Check console for details.</p>`;
            }
        };

        // --- TESTIMONIAL LOGIC ---

        const getTestimonialsCollection = () => {
            return collection(db, `artifacts/${appId}/public/data/testimonials`);
        };

        const setupTestimonialListener = () => {
            if (!db || !isFirebaseReady) return;

            const q = query(getTestimonialsCollection());
            
            onSnapshot(q, (snapshot) => {
                const testimonials = [];
                snapshot.forEach((doc) => {
                    testimonials.push({ id: doc.id, ...doc.data() });
                });
                renderTestimonials(testimonials);
            }, (error) => {
                console.error("Error listening to testimonials:", error);
            });
        };

        const renderTestimonials = (testimonials) => {
            const listElement = document.getElementById('testimonials-list');
            listElement.innerHTML = ''; 
            
            if (testimonials.length === 0) {
                listElement.innerHTML = `<p class="text-center col-span-full italic text-gray-400">Be the first to share your experience!</p>`;
                return;
            }

            testimonials.sort((a, b) => {
                const timeA = a.timestamp?.toDate ? a.timestamp.toDate().getTime() : 0;
                const timeB = b.timestamp?.toDate ? b.timestamp.toDate().getTime() : 0;
                return timeB - timeA;
            });

            testimonials.forEach(t => {
                const date = t.timestamp?.toDate ? t.timestamp.toDate().toLocaleDateString() : 'N/A';
                const div = document.createElement('div');
                div.className = 'bg-[#33261F] p-6 rounded-xl shadow-lg border border-orange-pearl/30 hover:shadow-2xl transition duration-300';
                div.innerHTML = `
                    <p class="text-gray-300 italic mb-4">"${t.message}"</p>
                    <div class="flex justify-between items-center text-sm text-orange-pearl font-semibold">
                        <span>— ${t.name || 'Anonymous Client'}</span>
                        <span class="text-xs text-gray-400">${date}</span>
                    </div>
                `;
                listElement.appendChild(div);
            });
        };
        
        window.submitTestimonial = async (event) => {
            event.preventDefault();
            if (!isFirebaseReady) return;

            const name = document.getElementById('testimonial-name').value.trim();
            const message = document.getElementById('testimonial-message').value.trim();
            const submitButton = document.getElementById('testimonial-submit-btn');
            const submitStatus = document.getElementById('testimonial-status');
            
            if (!message) {
                submitStatus.textContent = "Please enter a message.";
                submitStatus.classList.add('text-red-400');
                return;
            }

            submitButton.disabled = true;
            submitStatus.textContent = "Submitting...";
            submitStatus.classList.remove('text-red-400', 'text-green-400');
            
            try {
                await addDoc(getTestimonialsCollection(), {
                    name: name || 'Anonymous',
                    message: message,
                    timestamp: serverTimestamp(),
                    userId: userId, 
                });

                submitStatus.textContent = "Thank you for your testimonial! It is now live.";
                submitStatus.classList.add('text-green-400');
                document.getElementById('testimonial-form').reset();
            } catch (e) {
                console.error("Error adding testimonial: ", e);
                submitStatus.textContent = "Submission failed. See console for details.";
                submitStatus.classList.add('text-red-400');
            } finally {
                submitButton.disabled = false;
            }
        };

        // --- CALCULATOR LOGIC ---
        window.calculateCosts = () => {
            const type = document.getElementById('calc-type').value;
            const size = parseFloat(document.getElementById('calc-size').value);
            const quality = document.getElementById('calc-quality').value;
            const resultElement = document.getElementById('calc-result');

            if (isNaN(size) || size <= 0) {
                resultElement.innerHTML = "<p class='text-red-400 font-medium'>Please enter a valid size (sq ft) greater than zero.</p>";
                return;
            }

            let minCostPerSqFt, maxCostPerSqFt, label;

            if (type === 'new-home') {
                label = 'New Home Build';
                if (quality === 'low') {
                    // $150 - $250 per sq ft (Low End)
                    minCostPerSqFt = 150; maxCostPerSqFt = 250;
                } else { // high
                    // $250 - $450 per sq ft (High End)
                    minCostPerSqFt = 250; maxCostPerSqFt = 450;
                }
            } else { // renovation
                label = 'Major Renovation';
                if (quality === 'low') {
                    // $60 - $120 per sq ft (Low End)
                    minCostPerSqFt = 60; maxCostPerSqFt = 120;
                } else { // high
                    // $120 - $220 per sq ft (High End)
                    minCostPerSqFt = 120; maxCostPerSqFt = 220;
                }
            }

            const minCost = size * minCostPerSqFt;
            const maxCost = size * maxCostPerSqFt;

            const formatter = new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD', minimumFractionDigits: 0 });

            resultElement.innerHTML = `
                <div class="mt-4 p-4 bg-orange-pearl/10 rounded-lg border-l-4 border-orange-pearl">
                    <h4 class="text-lg font-bold text-light-text">${label} Estimate (${quality.toUpperCase()} End)</h4>
                    <p class="text-sm text-gray-300">Based on ${size.toLocaleString()} sq ft:</p>
                    <p class="text-3xl font-extrabold text-orange-pearl mt-2">
                        ${formatter.format(minCost)} - ${formatter.format(maxCost)}
                    </p>
                    <p class="text-xs text-gray-400 mt-2">*This is an estimate and does not include lot/land cost, permits, or specific luxury finishes. Contact us for a precise, detailed quote.</p>
                </div>
            `;
        };
        
        // --- INITIALIZATION ---
        initFirebase();
    </script>


    <!-- NAVIGATION -->
    <header class="sticky top-0 z-50 bg-dark-brown/95 backdrop-blur-sm shadow-xl">
        <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-4 flex justify-between items-center">
            
            <!-- Logo: Shirt You Not LLc (SYN LLC) -->
            <a href="#" class="flex items-center space-x-2">
                <div class="text-4xl font-extrabold tracking-widest text-orange-pearl text-shadow-orange">
                    SYN
                </div>
                <div class="flex flex-col text-xs leading-none">
                    <span class="text-light-text font-bold">SHIRT YOU NOT LLc</span>
                    <span class="text-gray-400">Design & Build</span>
                </div>
            </a>

            <!-- Desktop Nav Links -->
            <nav class="hidden md:flex space-x-8">
                <a href="#services" class="text-light-text hover:text-orange-pearl transition duration-200 font-medium">Services</a>
                <a href="#pricing" class="text-light-text hover:text-orange-pearl transition duration-200 font-medium">Pricing</a>
                <a href="#calculator" class="text-light-text hover:text-orange-pearl transition duration-200 font-medium">Calculator</a>
                <a href="#testimonials" class="text-light-text hover:text-orange-pearl transition duration-200 font-medium">Testimonials</a>
                <a href="#contact" class="text-light-text hover:text-orange-pearl transition duration-200 font-medium">Contact</a>
            </nav>

            <!-- Mobile Menu Placeholder -->
            <div class="md:hidden">
                <!-- Simple hamburger icon for mobile view -->
                <button class="text-light-text focus:outline-none">
                    <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16m-7 6h7"></path></svg>
                </button>
            </div>
        </div>
    </header>

    <main>
        <!-- HERO SECTION -->
        <section id="home" class="relative overflow-hidden pt-16 pb-20 md:pt-24 md:pb-32">
            <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 text-center">
                <h1 class="text-6xl md:text-8xl font-black leading-tight mb-4 text-light-text">
                    Build. Design. <span class="text-orange-pearl text-shadow-orange">Innovate.</span>
                </h1>
                <p class="text-xl md:text-2xl text-gray-300 max-w-3xl mx-auto mb-10">
                    Shirt You Not LLc delivers exceptional **Design Build and Renovation** services with quality and transparency.
                </p>
                <a href="#calculator" class="inline-block px-10 py-4 text-lg font-bold bg-orange-pearl text-dark-brown rounded-full shadow-2xl hover:bg-light-text hover:text-dark-brown transition duration-300 transform hover:scale-105">
                    Get an Instant Estimate
                </a>
            </div>
        </section>
        
        <!-- SERVICES SECTION -->
        <section id="services" class="py-20 bg-[#33261F] border-y border-orange-pearl/20">
            <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
                <h2 class="text-4xl font-bold text-center mb-12 text-light-text">Our Core Services</h2>
                <div class="grid md:grid-cols-2 gap-10">
                    <!-- Design Build -->
                    <div class="p-8 bg-dark-brown rounded-xl shadow-xl border-t-4 border-orange-pearl">
                        <div class="text-5xl text-orange-pearl mb-4">
                            <!-- Icon for Design Build -->
                            <svg class="w-12 h-12" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 11H5m14 0a2 2 0 012 2v6a2 2 0 01-2 2H5a2 2 0 01-2-2v-6a2 2 0 012-2m14 0V9a2 2 0 00-2-2M5 11V9a2 2 0 012-2m0 0V5a2 2 0 012-2h6a2 2 0 012 2v2M7 7h10"></path></svg>
                        </div>
                        <h3 class="text-2xl font-semibold mb-3 text-light-text">Design Build</h3>
                        <p class="text-gray-300">
                            A single point of contact for your entire project, from initial concept and architectural design to final construction and handover. We streamline the process, ensuring cost savings and quicker project completion.
                        </p>
                    </div>

                    <!-- Renovations -->
                    <div class="p-8 bg-dark-brown rounded-xl shadow-xl border-t-4 border-orange-pearl">
                        <div class="text-5xl text-orange-pearl mb-4">
                            <!-- Icon for Renovation -->
                            <svg class="w-12 h-12" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 21V3m-4 18h8M3 9h18m-6 0v10m-6-10v10"></path></svg>
                        </div>
                        <h3 class="text-2xl font-semibold mb-3 text-light-text">Renovations</h3>
                        <p class="text-gray-300">
                            Transform your existing space with our high-end and low-end renovation expertise. Whether it's a simple kitchen update or a complete home overhaul, we maximize your home's value and aesthetics.
                        </p>
                    </div>
                </div>
            </div>
        </section>

        <!-- PRICING SECTION -->
        <section id="pricing" class="py-20">
            <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
                <h2 class="text-4xl font-bold text-center mb-12 text-light-text">Current Project Pricing Guide</h2>
                
                <!-- New Home Pricing -->
                <h3 class="text-2xl font-semibold text-orange-pearl mb-6 border-b border-orange-pearl/30 pb-2">New Home Build (Per Sq. Ft. Range)</h3>
                <div class="grid md:grid-cols-2 gap-8 mb-12">
                    <div class="p-6 bg-[#33261F] rounded-xl border border-orange-pearl/50">
                        <h4 class="text-xl font-bold mb-2 text-light-text">Standard Quality Build (Low End)</h4>
                        <p class="text-4xl font-extrabold text-orange-pearl mb-4">$150 - $250</p>
                        <p class="text-gray-300">
                            Includes quality materials, standard finishes (e.g., laminate countertops, carpet/vinyl flooring), energy-efficient features, and a streamlined design process.
                        </p>
                    </div>
                    <div class="p-6 bg-[#33261F] rounded-xl border border-orange-pearl/80">
                        <h4 class="text-xl font-bold mb-2 text-light-text">Luxury Custom Build (High End)</h4>
                        <p class="text-4xl font-extrabold text-orange-pearl mb-4">$250 - $450+</p>
                        <p class="text-gray-300">
                            Custom architectural design, high-end finishes (e.g., solid wood flooring, marble/granite countertops, smart home integration), and premium appliance packages.
                        </p>
                    </div>
                </div>

                <!-- Renovation Pricing -->
                <h3 class="text-2xl font-semibold text-orange-pearl mb-6 border-b border-orange-pearl/30 pb-2">Major Renovation (Per Sq. Ft. Range)</h3>
                <div class="grid md:grid-cols-2 gap-8">
                    <div class="p-6 bg-[#33261F] rounded-xl border border-orange-pearl/50">
                        <h4 class="text-xl font-bold mb-2 text-light-text">Value Renovation (Low End)</h4>
                        <p class="text-4xl font-extrabold text-orange-pearl mb-4">$60 - $120</p>
                        <p class="text-gray-300">
                            Focuses on functional upgrades, cosmetic refresh (paint, hardware), and cost-effective material choices to maximize return on investment.
                        </p>
                    </div>
                    <div class="p-6 bg-[#33261F] rounded-xl border border-orange-pearl/80">
                        <h4 class="text-xl font-bold mb-2 text-light-text">Premium Renovation (High End)</h4>
                        <p class="text-4xl font-extrabold text-orange-pearl mb-4">$120 - $220+</p>
                        <p class="text-gray-300">
                            Full structural changes, custom cabinetry, luxury fixtures, and detailed design consultations for a completely transformed space.
                        </p>
                    </div>
                </div>
                <p class="text-center text-sm text-gray-400 mt-8">*All pricing is an estimate based on current market rates and complexity. Use the calculator below for an instant rough quote.</p>
            </div>
        </section>

        <!-- CALCULATOR SECTION -->
        <section id="calculator" class="py-20 bg-[#33261F] border-y border-orange-pearl/20">
            <div class="max-w-xl mx-auto px-4 sm:px-6 lg:px-8">
                <h2 class="text-4xl font-bold text-center mb-12 text-light-text">Cost Calculator</h2>
                
                <form onsubmit="event.preventDefault(); calculateCosts();" class="bg-dark-brown p-8 rounded-xl shadow-2xl space-y-6">
                    <div>
                        <label for="calc-type" class="block text-sm font-medium text-gray-300 mb-2">Project Type</label>
                        <select id="calc-type" class="w-full p-3 rounded-lg bg-[#4D392E] border border-orange-pearl/50 focus:border-orange-pearl focus:ring-1 focus:ring-orange-pearl text-light-text appearance-none cursor-pointer">
                            <option value="new-home">New Home Build</option>
                            <option value="renovation">Major Renovation</option>
                        </select>
                    </div>
                    <div>
                        <label for="calc-size" class="block text-sm font-medium text-gray-300 mb-2">Project Size (Square Footage)</label>
                        <input type="number" id="calc-size" min="50" placeholder="e.g., 2500" class="w-full p-3 rounded-lg bg-[#4D392E] border border-orange-pearl/50 focus:border-orange-pearl focus:ring-1 focus:ring-orange-pearl text-light-text" required>
                    </div>
                    <div>
                        <label for="calc-quality" class="block text-sm font-medium text-gray-300 mb-2">Desired Quality/Finish Level</label>
                        <select id="calc-quality" class="w-full p-3 rounded-lg bg-[#4D392E] border border-orange-pearl/50 focus:border-orange-pearl focus:ring-1 focus:ring-orange-pearl text-light-text appearance-none cursor-pointer">
                            <option value="low">Standard / Value (Low End)</option>
                            <option value="high">Custom / Luxury (High End)</option>
                        </select>
                    </div>
                    
                    <button type="submit" id="calc-submit-btn" class="w-full py-3 bg-orange-pearl text-dark-brown font-bold rounded-lg shadow-lg hover:bg-light-text transition duration-300 transform hover:scale-[1.01]">
                        Calculate Estimate
                    </button>
                </form>

                <!-- Calculation Result -->
                <div id="calc-result" class="mt-8 text-center text-light-text">
                    Enter your project details above to get an instant cost range.
                </div>
            </div>
        </section>

        <!-- TESTIMONIALS SECTION -->
        <section id="testimonials" class="py-20">
            <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
                <h2 class="text-4xl font-bold text-center mb-12 text-light-text">Client Testimonials</h2>
                
                <!-- Testimonials List (Real-time from Firestore) -->
                <div id="testimonials-list" class="grid md:grid-cols-2 lg:grid-cols-3 gap-8 mb-16">
                    <p class="text-center col-span-full italic text-gray-400">Loading testimonials...</p>
                </div>
                
                <!-- Submission Form -->
                <div id="testimonial-form-container" class="max-w-2xl mx-auto p-8 bg-[#33261F] rounded-xl shadow-2xl border-t-4 border-orange-pearl">
                    <h3 class="text-2xl font-bold mb-4 text-light-text text-center">Share Your Experience</h3>
                    <p class="text-center text-gray-400 mb-6">Leave a message below. We appreciate video reviews, but for the website, please submit text only. If you wish to submit a video, please contact us directly!</p>

                    <form id="testimonial-form" onsubmit="submitTestimonial(event);" class="space-y-4">
                        <div>
                            <label for="testimonial-name" class="block text-sm font-medium text-gray-300 mb-1">Your Name (Optional)</label>
                            <input type="text" id="testimonial-name" placeholder="John D." class="w-full p-3 rounded-lg bg-[#4D392E] border border-orange-pearl/50 text-light-text">
                        </div>
                        <div>
                            <label for="testimonial-message" class="block text-sm font-medium text-gray-300 mb-1">Your Testimonial Message*</label>
                            <textarea id="testimonial-message" rows="4" placeholder="Shirt You Not LLC did an amazing job on our home renovation..." class="w-full p-3 rounded-lg bg-[#4D392E] border border-orange-pearl/50 text-light-text" required></textarea>
                        </div>
                        <div id="testimonial-status" class="text-center text-sm font-medium h-5"></div>
                        <button type="submit" id="testimonial-submit-btn" class="w-full py-3 bg-orange-pearl text-dark-brown font-bold rounded-lg shadow-lg hover:bg-light-text transition duration-300 transform hover:scale-[1.01]">
                            Submit Review
                        </button>
                    </form>
                </div>
            </div>
        </section>
    </main>

    <!-- FOOTER/CONTACT SECTION -->
    <footer id="contact" class="bg-dark-brown pt-16 pb-8 border-t border-orange-pearl/30">
        <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
            <h2 class="text-4xl font-bold text-center mb-10 text-light-text">Contact Raymond Brown</h2>
            
            <div class="grid md:grid-cols-3 gap-10 text-center">
                <div class="p-4 rounded-lg bg-[#33261F]">
                    <p class="text-sm text-gray-400">Owner</p>
                    <p class="text-2xl font-bold text-orange-pearl">Raymond Brown</p>
                </div>
                <div class="p-4 rounded-lg bg-[#33261F]">
                    <p class="text-sm text-gray-400">Phone Contact</p>
                    <a href="tel:8502084444" class="text-2xl font-bold text-light-text hover:text-orange-pearl transition duration-200">850-208-4444</a>
                </div>
                <div class="p-4 rounded-lg bg-[#33261F]">
                    <p class="text-sm text-gray-400">Email Address</p>
                    <a href="mailto:synraybrown@gmail.com" class="text-xl font-bold text-light-text hover:text-orange-pearl transition duration-200">synraybrown@gmail.com</a>
                </div>
            </div>

            <div class="text-center mt-16 border-t border-gray-700 pt-8">
                <p class="text-gray-500 text-sm">&copy; 2024 Shirt You Not LLc. All rights reserved.</p>
                <p class="text-gray-500 text-xs mt-1">Built with Design Build & Renovation Excellence.</p>
            </div>
        </div>
    </footer>

</body>
</html>
