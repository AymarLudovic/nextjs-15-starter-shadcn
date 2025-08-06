"use client"

import { useState } from "react"
import { Globe, ArrowUp, Copy, Download } from 'lucide-react'

type AnimationFile = {
  url: string
  content: string
  type: "css" | "js"
  isAnimation: boolean
  library?: string
  confidence: number // Score de confiance 0-100
}

type Result = {
  title: string
  description: string
  techGuesses: string[]
  internalLinks: number
  externalLinks: number
  images: string[]
  stylesheets: number
  openGraphTags: number
  fullHTML: string
  fullCSS: string
  fullJS: string
  baseURL: string
  animationFiles: AnimationFile[]
  requiredCdnUrls: string[]
}

// Component for each result item
const ResultItem = ({ label, value }: { label: string; value: string | number }) => (
  <div className="flex justify-between items-center py-4 border-b border-gray-800/50">
    <p className="text-gray-400">{label}</p>
    <p className="text-[#e4e4e4] text-right font-medium truncate pl-4">{value}</p>
  </div>
)

export default function SiteInspector() {
  const [url, setUrl] = useState("")
  const [result, setResult] = useState<Result | null>(null)
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)
  const [copyStatus, setCopyStatus] = useState<{ id: string; message: string } | null>(null)

  const proposalUrls = ["cosmos.so", "stripe.com", "linear.app"]

  const MAX_FILE_FETCH_ATTEMPTS = 15

  // Helper function for fetching with retry - IMPROVED VERSION
  const fetchWithRetry = async (
    url: string,
    maxAttempts: number = MAX_FILE_FETCH_ATTEMPTS,
    delay = 1000,
  ): Promise<{ success: boolean; content: string; error?: string }> => {
    for (let attempt = 1; attempt <= maxAttempts; attempt++) {
      try {
        const proxyUrl = `https://api.allorigins.win/get?url=${encodeURIComponent(url)}`
        console.log(`🔄 Attempt ${attempt}/${maxAttempts} for: ${url}`)
        
        const response = await fetch(proxyUrl)
        if (!response.ok) {
          throw new Error(`Network response was not ok (status: ${response.status})`)
        }
        const data = await response.json()
        if (!data.contents) {
          throw new Error(`No content received from proxy`)
        }
        
        console.log(`✅ SUCCESS on attempt ${attempt}/${maxAttempts} for: ${url}`)
        return { success: true, content: data.contents }
      } catch (error) {
        const errorMsg = error instanceof Error ? error.message : String(error)
        console.warn(`⚠️ FAILED attempt ${attempt}/${maxAttempts} for ${url}: ${errorMsg}`)
        
        if (attempt < maxAttempts) {
          console.log(`⏳ Waiting ${delay}ms before retry...`)
          await new Promise((resolve) => setTimeout(resolve, delay))
        } else {
          console.error(`❌ FINAL FAILURE after ${maxAttempts} attempts for: ${url}`)
          return { success: false, content: "", error: errorMsg }
        }
      }
    }
    return { success: false, content: "", error: "Should not reach here" }
  }

  // 🔗 CDN Links pour les bibliothèques d'animation
  const getLibraryCDN = (library: string): string[] => {
    const cdnMap: { [key: string]: string[] } = {
      GSAP: [
        "https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.2/gsap.min.js",
        "https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.2/ScrollTrigger.min.js",
        "https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.2/TextPlugin.min.js",
        "https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.2/MotionPathPlugin.min.js",
      ],
      "Three.js": [
        "https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js",
        "https://cdnjs.cloudflare.com/ajax/libs/dat-gui/0.7.9/dat.gui.min.js",
      ],
      Lottie: ["https://cdnjs.cloudflare.com/ajax/libs/lottie-web/5.12.2/lottie.min.js"],
      AOS: [
        "https://cdnjs.cloudflare.com/ajax/libs/aos/2.3.4/aos.js",
        "https://cdnjs.cloudflare.com/ajax/libs/aos/2.3.4/aos.css",
      ],
      "Anime.js": ["https://cdnjs.cloudflare.com/ajax/libs/animejs/3.2.1/anime.min.js"],
      "Locomotive Scroll": [
        "https://cdn.jsdelivr.net/npm/locomotive-scroll@4.1.4/dist/locomotive-scroll.min.js",
        "https://cdn.jsdelivr.net/npm/locomotive-scroll@4.1.4/dist/locomotive-scroll.min.css",
      ],
      "Barba.js": ["https://cdnjs.cloudflare.com/ajax/libs/barba.js/1.0.0/barba.min.js"],
      ScrollMagic: [
        "https://cdnjs.cloudflare.com/ajax/libs/ScrollMagic/2.0.8/ScrollMagic.min.js",
        "https://cdnjs.cloudflare.com/ajax/libs/ScrollMagic/2.0.8/plugins/animation.gsap.min.js",
      ],
      "Velocity.js": ["https://cdnjs.cloudflare.com/ajax/libs/velocity/2.0.6/velocity.min.js"],
      Swiper: [
        "https://cdn.jsdelivr.net/npm/swiper@8/swiper-bundle.min.js",
        "https://cdn.jsdelivr.net/npm/swiper@8/swiper-bundle.min.css",
      ],
      Particles: ["https://cdn.jsdelivr.net/npm/particles.js@2.0.0/particles.min.js"],
    }

    return cdnMap[library] || []
  }

  // 🎯 Fonction améliorée pour détecter les bibliothèques d'animation avec score de confiance
  const detectAnimationLibrary = (
    url: string,
    content: string,
  ): { isAnimation: boolean; library?: string; confidence: number } => {
    const urlLower = url.toLowerCase()
    const contentLower = content.toLowerCase()

    // ❌ BLACKLIST - Exclure les faux positifs
    const blacklist = [
      "googletagmanager",
      "google-analytics",
      "gtag",
      "facebook.net",
      "doubleclick",
      "adsystem",
      "googlesyndication",
      "hotjar",
      "intercom",
      "zendesk",
      "crisp.chat",
      "tawk.to",
    ]

    const isBlacklisted = blacklist.some((item) => urlLower.includes(item))
    if (isBlacklisted) {
      return { isAnimation: false, confidence: 0 }
    }

    const libraries = [
      {
        name: "GSAP",
        patterns: [
          { pattern: /gsap\.registerPlugin|gsap\.timeline|gsap\.to|gsap\.from/gi, weight: 95 },
          { pattern: /greensock|tweenmax|tweenlite|timelinemax/gi, weight: 90 },
          { pattern: /scrolltrigger|motionpath|drawsvg/gi, weight: 85 },
          { pattern: /gsap/gi, weight: 70 },
        ],
      },
      {
        name: "Three.js",
        patterns: [
          { pattern: /new THREE\.|THREE\.Scene|THREE\.WebGLRenderer/gi, weight: 95 },
          { pattern: /PerspectiveCamera|BufferGeometry|MeshBasicMaterial/gi, weight: 90 },
          { pattern: /three\.js|three\.min\.js/gi, weight: 85 },
          { pattern: /webgl|canvas.*3d/gi, weight: 60 },
        ],
      },
      {
        name: "Lottie",
        patterns: [
          { pattern: /lottie\.loadAnimation|bodymovin/gi, weight: 95 },
          { pattern: /lottie-web|lottie\.js/gi, weight: 85 },
          { pattern: /lottie/gi, weight: 70 },
        ],
      },
      {
        name: "AOS",
        patterns: [
          { pattern: /AOS\.init|data-aos/gi, weight: 95 },
          { pattern: /aos\.js|animate.*on.*scroll/gi, weight: 85 },
        ],
      },
      {
        name: "Anime.js",
        patterns: [
          { pattern: /anime\(\{|anime\.timeline/gi, weight: 95 },
          { pattern: /anime\.js|animejs/gi, weight: 85 },
        ],
      },
      {
        name: "Locomotive Scroll",
        patterns: [
          { pattern: /new LocomotiveScroll|data-scroll/gi, weight: 95 },
          { pattern: /locomotive-scroll/gi, weight: 85 },
        ],
      },
      {
        name: "Framer Motion",
        patterns: [{ pattern: /framer-motion|motion\.|useAnimation|AnimatePresence/gi, weight: 95 }],
      },
    ]

    let bestMatch = { library: "", confidence: 0 }

    for (const lib of libraries) {
      let totalScore = 0
      let matchCount = 0

      for (const { pattern, weight } of lib.patterns) {
        const matches = (urlLower + " " + contentLower).match(pattern)
        if (matches) {
          totalScore += weight * matches.length
          matchCount++
        }
      }

      if (matchCount > 0) {
        const confidence = Math.min(100, totalScore / matchCount)
        if (confidence > bestMatch.confidence) {
          bestMatch = { library: lib.name, confidence }
        }
      }
    }

    // Détection générique d'animations avec score plus bas
    if (bestMatch.confidence === 0) {
      const genericPatterns = [
        /@keyframes|animation:|transform:|transition:/gi,
        /requestAnimationFrame|setInterval.*animation/gi,
        /\.animate\(|\.transition\(/gi,
        /transform.*translate|rotate|scale/gi,
        /opacity.*transition|visibility.*transition/gi,
        /cubic-bezier|ease-in|ease-out/gi,
      ]

      let genericScore = 0
      for (const pattern of genericPatterns) {
        const matches = contentLower.match(pattern)
        if (matches) {
          genericScore += matches.length * 10
        }
      }

      if (genericScore > 20) {
        return { isAnimation: true, confidence: Math.min(50, genericScore) }
      }
    }

    return {
      isAnimation: bestMatch.confidence > 60,
      library: bestMatch.library || undefined,
      confidence: bestMatch.confidence,
    }
  }

  const analyzeSite = async (urlToAnalyze = url) => {
    if (!urlToAnalyze) return

    setLoading(true)
    setError(null)
    setResult(null)
    setCopyStatus(null)

    try {
      // Analyse basique existante mais améliorée
      const MAX_SITE_FETCH_ATTEMPTS = 10
      for (let attempt = 1; attempt <= MAX_SITE_FETCH_ATTEMPTS; attempt++) {
        try {
          let fullUrl = urlToAnalyze
          if (!/^https?:\/\//i.test(fullUrl)) {
            fullUrl = "https://" + fullUrl
          }

          console.log(`🔍 Analyzing: ${fullUrl} (Attempt ${attempt}/${MAX_SITE_FETCH_ATTEMPTS})`)

          const response = await fetch(`https://api.allorigins.win/get?url=${encodeURIComponent(fullUrl)}`)
          if (!response.ok) {
            throw new Error(`Network response was not ok (status: ${response.status})`)
          }

          const data = await response.json()
          const html = data.contents
          const parser = new DOMParser()
          const doc = parser.parseFromString(html, "text/html")
          const baseURL = new URL(fullUrl).origin

          console.log(`📄 HTML parsed, base URL: ${baseURL}`)

          const title = doc.querySelector("title")?.textContent || "No title found"
          const description = doc.querySelector('meta[name="description"]')?.getAttribute("content") || "Not found"

          // Analyse des liens
          const links = Array.from(doc.querySelectorAll("a[href]")).map((el) => el.getAttribute("href") || "")
          const internalLinks = links.filter((href) => {
            try {
              return new URL(href, baseURL).hostname === new URL(fullUrl).hostname
            } catch (e) {
              return false
            }
          }).length
          const externalLinks = links.length - internalLinks

          // Images
          const imageEls = Array.from(doc.querySelectorAll("img"))
          const imageSrcs: string[] = imageEls
            .map((img) => {
              const rawSrc = img.getAttribute("src")
              if (!rawSrc) return null
              try {
                return new URL(rawSrc, baseURL).href
              } catch (e) {
                return null
              }
            })
            .filter((src): src is string => !!src)

          const ogTags = doc.querySelectorAll('meta[property^="og:"]').length

          // 🎨 RÉCUPÉRATION AMÉLIORÉE DES CSS
          console.log("🎨 Fetching CSS files...")
          const stylesheetLinks = Array.from(doc.querySelectorAll('link[rel="stylesheet"]'))
            .map((link) => link.getAttribute("href"))
            .filter(Boolean) as string[]

          console.log(`📋 Found ${stylesheetLinks.length} CSS files:`, stylesheetLinks)

          const animationFiles: AnimationFile[] = []
          const cssFiles: string[] = []

          // Récupération des CSS externes avec détection améliorée - IMPROVED LOGIC
          for (const href of stylesheetLinks) {
            const fullHref = new URL(href, baseURL).href
            console.log(`🎨 Starting CSS fetch for: ${fullHref}`)
            
            const fetchResult = await fetchWithRetry(fullHref)
            
            if (fetchResult.success) {
              console.log(`✅ CSS successfully fetched: ${fullHref}`)
              const animationInfo = detectAnimationLibrary(fullHref, fetchResult.content)

              if (animationInfo.isAnimation && animationInfo.confidence > 60) {
                console.log(
                  `✨ CSS animation file detected: ${fullHref} (${animationInfo.library || "Generic"}) - Confidence: ${animationInfo.confidence}%`,
                )
                animationFiles.push({
                  url: fullHref,
                  content: fetchResult.content,
                  type: "css",
                  isAnimation: true,
                  library: animationInfo.library,
                  confidence: animationInfo.confidence,
                })
              }

              cssFiles.push(
                `/* Fetched from: ${fullHref} ${animationInfo.isAnimation ? `(ANIMATION FILE - ${animationInfo.confidence}%)` : ""} */\n${fetchResult.content}`,
              )
            } else {
              console.error(`❌ CSS fetch completely failed for: ${href} - Error: ${fetchResult.error}`)
              cssFiles.push(`/* FETCH FAILED after ${MAX_FILE_FETCH_ATTEMPTS} attempts: ${href} - Error: ${fetchResult.error} */`)
            }
          }

          // Récupération des CSS inline
          const inlineCSS = Array.from(doc.querySelectorAll("style")).map((el, index) => {
            const content = el.textContent || ""
            const animationInfo = detectAnimationLibrary(`inline-style-${index}`, content)

            if (animationInfo.isAnimation && animationInfo.confidence > 60) {
              console.log(
                `✨ Inline CSS animation detected (style ${index}) - Confidence: ${animationInfo.confidence}%`,
              )
              animationFiles.push({
                url: `inline-style-${index}`,
                content: content,
                type: "css",
                isAnimation: true,
                library: animationInfo.library,
                confidence: animationInfo.confidence,
              })
            }

            return `/* Inline style ${index} ${animationInfo.isAnimation ? `(ANIMATION - ${animationInfo.confidence}%)` : ""} */\n${content}`
          })

          const fullCSS = [...cssFiles, ...inlineCSS].join("\n\n")

          // 🚀 RÉCUPÉRATION AMÉLIORÉE DES JAVASCRIPT
          console.log("🚀 Analyzing JavaScript files...")

          const scriptEls = Array.from(doc.querySelectorAll("script")).map((el) => ({
            src: el.getAttribute("src"),
            content: el.textContent,
            type: el.getAttribute("type") || "text/javascript",
          }))

          console.log(`📋 Found ${scriptEls.length} script elements`)

          const jsFiles: string[] = []

          // 1. Récupération des JS externes avec filtrage amélioré - IMPROVED LOGIC
          const externalScripts = scriptEls.filter((script) => !!script.src)
          console.log(`🔗 External scripts to fetch: ${externalScripts.length}`)

          for (const script of externalScripts) {
            const fullSrc = new URL(script.src!, baseURL).href
            console.log(`🚀 Starting JS fetch for: ${fullSrc}`)
            
            const fetchResult = await fetchWithRetry(fullSrc)
            
            if (fetchResult.success) {
              console.log(`✅ JS successfully fetched: ${fullSrc}`)
              const animationInfo = detectAnimationLibrary(fullSrc, fetchResult.content)

              if (animationInfo.isAnimation && animationInfo.confidence > 60) {
                console.log(
                  `🎭 JS animation file detected: ${fullSrc} (${animationInfo.library || "Generic"}) - Confidence: ${animationInfo.confidence}%`,
                )
                animationFiles.push({
                  url: fullSrc,
                  content: fetchResult.content,
                  type: "js",
                  isAnimation: true,
                  library: animationInfo.library,
                  confidence: animationInfo.confidence,
                })
              } else if (animationInfo.confidence > 0) {
                console.log(`⚠️ Low confidence animation: ${fullSrc} - Confidence: ${animationInfo.confidence}%`)
              }

              jsFiles.push(
                `// Fetched from: ${fullSrc} ${animationInfo.isAnimation ? `(ANIMATION FILE - ${animationInfo.confidence}%)` : ""}\n// Library: ${animationInfo.library || "None"}\n${fetchResult.content}`,
              )
            } else {
              console.error(`❌ JS fetch completely failed for: ${script.src} - Error: ${fetchResult.error}`)
              jsFiles.push(`// FETCH FAILED after ${MAX_FILE_FETCH_ATTEMPTS} attempts: ${script.src} - Error: ${fetchResult.error}`)
            }
          }

          // 2. Récupération des JS inline
          const inlineScripts = scriptEls.filter((script) => !script.src && script.content)
          console.log(`📝 Inline scripts to process: ${inlineScripts.length}`)

          const inlineJS = inlineScripts.map((script, index) => {
            const content = script.content || ""
            const animationInfo = detectAnimationLibrary(`inline-script-${index}`, content)

            if (animationInfo.isAnimation && animationInfo.confidence > 60) {
              console.log(
                `✨ Inline animation script detected (script ${index}) - Confidence: ${animationInfo.confidence}%`,
              )
              animationFiles.push({
                url: `inline-script-${index}`,
                content: content,
                type: "js",
                isAnimation: true,
                library: animationInfo.library,
                confidence: animationInfo.confidence,
              })
            }

            return `// Inline script ${index} ${animationInfo.isAnimation ? `(ANIMATION CODE - ${animationInfo.confidence}%)` : ""}\n// Library: ${animationInfo.library || "None"}\n${content}`
          })

          const fullJS = [...jsFiles, ...inlineJS].join("\n\n// ===== NEXT SCRIPT =====\n\n")

          console.log(`📊 Total scripts processed: ${externalScripts.length + inlineScripts.length}`)
          console.log(`🎭 High-confidence animation files found: ${animationFiles.length}`)

          // 🔗 COLLECTE DES URLS CDN POUR L'IFRAME
          const detectedLibraries = [...new Set(animationFiles.map((f) => f.library).filter(Boolean))]
          const allCdnUrls: string[] = []
          detectedLibraries.forEach((library) => {
            const cdnUrls = getLibraryCDN(library)
            allCdnUrls.push(...cdnUrls)
          })
          console.log(`🔗 Collected CDN URLs for iframe:`, allCdnUrls)

          // 🏗️ RECONSTRUCTION AMÉLIORÉE DU HTML
          const docClone = parser.parseFromString(html, "text/html")
          const bodyHTML = docClone.body.innerHTML
          const bodyAttributes = Array.from(docClone.body.attributes)
            .map((attr) => `${attr.name}="${attr.value}"`)
            .join(" ")

          const cleanedHTML = bodyAttributes ? `<body ${bodyAttributes}>${bodyHTML}</body>` : bodyHTML

          // 🎯 DÉTECTION DES TECHNOLOGIES
          const allCode = [fullJS, fullCSS, html].join(" ")
          const techGuesses = []

          const techPatterns = {
            React: /react|jsx|createelement/gi,
            Vue: /vue\.js|v-if|v-for|\{\{.*\}\}/gi,
            Angular: /angular|ng-|@component/gi,
            jQuery: /jquery|\$\(/gi,
            GSAP: /gsap|greensock|tweenmax|tweenlite/gi,
            "Framer Motion": /framer-motion|motion\./gi,
            Lottie: /lottie|bodymovin/gi,
            "Three.js": /three\.js|webgl/gi,
            Bootstrap: /bootstrap/gi,
            Tailwind: /tailwind/gi,
            AOS: /aos\.js|data-aos/gi,
            "Locomotive Scroll": /locomotive-scroll/gi,
            "Barba.js": /barba\.js/gi,
            Swiper: /swiper/gi,
            Particles: /particles/gi,
          }

          Object.entries(techPatterns).forEach(([tech, pattern]) => {
            if (pattern.test(allCode)) {
              techGuesses.push(tech)
            }
          })

          // Ajouter les bibliothèques détectées dans les fichiers d'animation
          animationFiles.forEach((file) => {
            if (file.library && !techGuesses.includes(file.library)) {
              techGuesses.push(file.library)
            }
          })

          console.log(`🎯 Technologies detected:`, techGuesses)

          setResult({
            title,
            description,
            techGuesses,
            internalLinks,
            externalLinks,
            images: imageSrcs,
            stylesheets: stylesheetLinks.length,
            openGraphTags: ogTags,
            fullHTML: cleanedHTML,
            fullCSS,
            fullJS,
            baseURL,
            animationFiles,
            requiredCdnUrls: allCdnUrls,
          })

          console.log(`✅ Analysis complete for: ${fullUrl}`)
          setLoading(false)
          return
        } catch (err) {
          console.warn(`⚠️ Main site analysis attempt ${attempt} failed:`, err)
          if (attempt === MAX_SITE_FETCH_ATTEMPTS) {
            setError("⌐ Analysis failed after multiple attempts.")
            setLoading(false)
          } else {
            await new Promise((res) => setTimeout(res, 2000))
          }
        }
      }
    } catch (err) {
      console.error("❌ Analysis failed:", err)
      setError(`Analysis failed: ${err instanceof Error ? err.message : String(err)}`)
      setLoading(false)
    }
  }

  const handleProposalClick = async (proposalUrl: string) => {
    setUrl(proposalUrl)
    await analyzeSite(proposalUrl)
  }

  const createDownloadLink = (content: string, filename: string, mimeType: string) => {
    const blob = new Blob([content], { type: mimeType })
    const url = URL.createObjectURL(blob)
    const a = document.createElement("a")
    a.href = url
    a.download = filename
    document.body.appendChild(a)
    a.click()
    document.body.removeChild(a)
    URL.revokeObjectURL(url)
  }

  const copyToClipboard = async (text: string, id: string) => {
    try {
      await navigator.clipboard.writeText(text)
      setCopyStatus({ id, message: "Copied! ✅" })
      setTimeout(() => setCopyStatus(null), 2000)
    } catch (err) {
      setCopyStatus({ id, message: "Copy Failed ⌐" })
      setTimeout(() => setCopyStatus(null), 2000)
    }
  }

  const handleCopyPrompt = () => {
    if (!result) return
    const prompt = `Here is the HTML and CSS code representing the pixel-perfect design of the website I want to create. Adapt it completely to the specified framework (e.g., React, Svelte, Vue). Do not alter the design or omit any elements.

IMPORTANT: This site uses the following animation libraries: ${
      result.animationFiles
        .map((f) => f.library)
        .filter(Boolean)
        .join(", ") || "None detected"
    }

High-confidence animations detected: ${result.animationFiles.filter((f) => f.confidence > 80).length}

Here are the CDN URLs for these libraries:
${result.requiredCdnUrls.join("\n")}

Here is the HTML code:
\`\`\`html
${result.fullHTML}
\`\`\`

Here is the CSS code:
\`\`\`css
${result.fullCSS}
\`\`\`

Here is the JavaScript code (including animations):
\`\`\`javascript
${result.fullJS}
\`\`\``
    copyToClipboard(prompt, "prompt")
  }

  const handleDownloadPrompt = () => {
    if (!result) return
    const promptContent = `Here is the HTML and CSS code representing the pixel-perfect design of the website I want to create. Adapt it completely to the specified framework (e.g., React, Svelte, Vue). Do not alter the design or omit any elements.

IMPORTANT: This site uses the following animation libraries: ${
      result.animationFiles
        .map((f) => f.library)
        .filter(Boolean)
        .join(", ") || "None detected"
    }

High-confidence animations detected: ${result.animationFiles.filter((f) => f.confidence > 80).length}

Here are the CDN URLs for these libraries:
${result.requiredCdnUrls.join("\n")}

Here is the HTML code:
\`\`\`html
${result.fullHTML}
\`\`\`

Here is the CSS code:
\`\`\`css
${result.fullCSS}
\`\`\`

Here is the JavaScript code (including animations):
\`\`\`javascript
${result.fullJS}
\`\`\``
    createDownloadLink(promptContent, "design_prompt.txt", "text/plain")
  }

  // 🎬 FONCTION POUR CRÉER LA PRÉVISUALISATION
  const createOptimizedPreview = () => {
    if (!result) return ""

    const cdnTags = result.requiredCdnUrls
      .map((url) => {
        if (url.endsWith(".css")) {
          return `    <link rel="stylesheet" href="${url}" crossorigin="anonymous">`
        } else {
          return `    <script src="${url}" crossorigin="anonymous"></script>`
        }
      })
      .join("\n")

    const animationCSS = result.animationFiles
      .filter((f) => f.type === "css")
      .map((f) => f.content)
      .join("\n\n")

    const animationJS = result.animationFiles
      .filter((f) => f.type === "js")
      .map((f) => f.content)
      .join("\n\n")

    const animationInitScript = `
// 🎬 ENHANCED ANIMATION INITIALIZATION
console.log('🎬 Initializing animations...');
console.log('🔗 CDN URLs loaded:', ${JSON.stringify(result.requiredCdnUrls)});
console.log('📊 High-confidence animations:', ${result.animationFiles.filter((f) => f.confidence > 80).length});

async function initializeAnimations() {
  console.log('🔄 Starting enhanced animation initialization...');
  
  // Wait for libraries to load
  await new Promise(resolve => setTimeout(resolve, 2000));
  
  // 1. GSAP
  if (typeof gsap !== 'undefined') {
    console.log('✅ GSAP ready - initializing animations...');
    
    try {
      gsap.set("*", {clearProps: "all"});
      
      const elementsToAnimate = document.querySelectorAll('h1, h2, h3, .hero, .title, [class*="fade"], [class*="slide"], [class*="animate"]');
      
      if (elementsToAnimate.length > 0) {
        gsap.from(elementsToAnimate, {
          opacity: 0,
          y: 50,
          duration: 1,
          stagger: 0.1,
          ease: "power2.out"
        });
      }
      
      if (typeof ScrollTrigger !== 'undefined') {
        gsap.registerPlugin(ScrollTrigger);
        
        gsap.utils.toArray('[data-scroll], .scroll-trigger').forEach(element => {
          gsap.from(element, {
            opacity: 0,
            y: 100,
            duration: 1,
            scrollTrigger: {
              trigger: element,
              start: "top 80%",
              end: "bottom 20%",
              toggleActions: "play none none reverse"
            }
          });
        });
      }
    } catch (e) {
      console.warn('⚠️ GSAP initialization error:', e);
    }
  }
  
  // 2. Three.js
  if (typeof THREE !== 'undefined') {
    console.log('✅ Three.js ready - initializing scene...');
    
    const canvas = document.querySelector('canvas') || document.querySelector('#three-canvas');
    if (canvas) {
      try {
        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, canvas.clientWidth / canvas.clientHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ canvas: canvas, alpha: true });
        
        renderer.setSize(canvas.clientWidth, canvas.clientHeight);
        
        const geometry = new THREE.BufferGeometry();
        const vertices = [];
        
        for (let i = 0; i < 1000; i++) {
          vertices.push(
            (Math.random() - 0.5) * 2000,
            (Math.random() - 0.5) * 2000,
            (Math.random() - 0.5) * 2000
          );
        }
        
        geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3));
        
        const material = new THREE.PointsMaterial({ color: 0xffffff, size: 2 });
        const particles = new THREE.Points(geometry, material);
        scene.add(particles);
        
        camera.position.z = 1000;
        
        function animate() {
          requestAnimationFrame(animate);
          particles.rotation.x += 0.001;
          particles.rotation.y += 0.001;
          renderer.render(scene, camera);
        }
        
        animate();
        console.log('✅ Three.js scene initialized');
      } catch (e) {
        console.warn('⚠️ Three.js initialization failed:', e);
      }
    }
  }
  
  // 3. AOS
  if (typeof AOS !== 'undefined') {
    console.log('✅ AOS ready - initializing...');
    try {
      AOS.init({
        duration: 1000,
        once: false,
        mirror: true,
        offset: 100
      });
    } catch (e) {
      console.warn('⚠️ AOS initialization error:', e);
    }
  }
  
  // 4. Lottie
  if (typeof lottie !== 'undefined') {
    console.log('✅ Lottie ready - initializing...');
    try {
      document.querySelectorAll('[data-lottie], .lottie, [data-animation-path]').forEach(el => {
        const path = el.dataset.lottie || el.dataset.animationPath;
        if (path) {
          lottie.loadAnimation({
            container: el,
            renderer: 'svg',
            loop: true,
            autoplay: true,
            path: path
          });
        }
      });
    } catch (e) {
      console.warn('⚠️ Lottie initialization error:', e);
    }
  }
  
  console.log('✅ Enhanced animation initialization complete');
}

// Initialize when DOM is ready
if (document.readyState === 'loading') {
  document.addEventListener('DOMContentLoaded', initializeAnimations);
} else {
  initializeAnimations();
}
`

    const previewHtml = `<!DOCTYPE html>
<html lang="en">
<head>
    <base href="${result.baseURL}">
    <title>UI Preview with Enhanced Animation Detection</title>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    
    <!-- CDN Libraries for Animations -->
${cdnTags}
    
    <!-- Animation CSS -->
    <style id="animation-styles">
${animationCSS}

/* Enhanced Animation CSS helpers */
.animate-in { animation-play-state: running !important; }
body.animations-loaded .fade-in { animation: fadeIn 1s ease-in-out; }
body.animations-loaded .slide-up { animation: slideUp 1s ease-out; }

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(20px); }
  to { opacity: 1; transform: translateY(0); }
}

@keyframes slideUp {
  from { opacity: 0; transform: translateY(50px); }
  to { opacity: 1; transform: translateY(0); }
}

* { animation-fill-mode: both; }
canvas { display: block; width: 100%; height: 100%; }
    </style>
    
    <!-- Regular CSS -->
    <style id="regular-styles">
${result.fullCSS}
    </style>
</head>
<body>
${result.fullHTML}

<!-- Enhanced Animation Initialization Script -->
<script>
${animationInitScript}
</script>

<!-- Animation JavaScript -->
<script>
${animationJS}
</script>

<!-- Regular JavaScript -->
<script>
${result.fullJS}
</script>

<script>
console.log('🎬 Enhanced preview ready');
console.log('📊 Analysis method: Basic');
console.log('🎭 Animation files detected:', ${result.animationFiles.length});
console.log('🎯 High-confidence animations:', ${result.animationFiles.filter((f) => f.confidence > 80).length});
</script>
</body>
</html>`

    return previewHtml
  }

  // 🎨 Fonction pour analyser la qualité des animations récupérées
  const analyzeAnimationQuality = () => {
    if (!result) return null

    const highConfidenceAnimations = result.animationFiles.filter((f) => f.confidence > 80)
    const mediumConfidenceAnimations = result.animationFiles.filter((f) => f.confidence > 60 && f.confidence <= 80)

    const animationStats = {
      totalFiles: result.animationFiles.length,
      highConfidence: highConfidenceAnimations.length,
      mediumConfidence: mediumConfidenceAnimations.length,
      cssAnimations: result.animationFiles.filter((f) => f.type === "css").length,
      jsAnimations: result.animationFiles.filter((f) => f.type === "js").length,
      libraries: [...new Set(result.animationFiles.map((f) => f.library).filter(Boolean))],
      hasInlineAnimations: result.animationFiles.some((f) => f.url.includes("inline")),
      estimatedCompleteness: Math.min(
        100,
        highConfidenceAnimations.length * 30 + mediumConfidenceAnimations.length * 15,
      ),
    }

    return animationStats
  }

  // 🔍 Fonction pour extraire uniquement les animations
  const extractAnimationsOnly = () => {
    if (!result) return { css: "", js: "" }

    const animationCSS = result.animationFiles
      .filter((f) => f.type === "css")
      .map((f) => `/* ${f.library || "Animation"} - ${f.url} - Confidence: ${f.confidence}% */\n${f.content}`)
      .join("\n\n")

    const animationJS = result.animationFiles
      .filter((f) => f.type === "js")
      .map((f) => `// ${f.library || "Animation"} - ${f.url} - Confidence: ${f.confidence}%\n${f.content}`)
      .join("\n\n")

    return { css: animationCSS, js: animationJS }
  }

  return (
    <div className="min-h-screen bg-black text-[#e4e4e4] p-4 sm:p-8" style={{ fontFamily: "Funnel Display" }}>
      <header className="max-w-4xl mx-auto flex justify-between items-center mb-12">
        <svg width="36" height="36" viewBox="0 0 32 32" xmlns="http://www.w3.org/2000/svg" fill="#e4e4e4">
          <rect x="0" y="0" width="32" height="32" rx="10" />
        </svg>
        <a
          href="/signup"
          className="h-[35px] w-auto px-5 flex items-center justify-center rounded-[13px] bg-[#e4e4e4] text-black font-semibold transition-opacity hover:opacity-90"
        >
          Sign up
        </a>
      </header>

      <div className="max-w-4xl mx-auto p-6 sm:p-10 pb-20">
        <div className="text-center mb-10">
          <h1 className="text-4xl sm:text-5xl font-bold text-[#e4e4e4] mb-4">Clone your favorite website design.</h1>
          <p className="text-lg text-gray-400 max-w-xl mx-auto">
            Paste a URL, launch the process, and instantly get a pixel-perfect replica of any website's design.
          </p>
        </div>

        <div className="h-[45px] w-[90%] sm:w-[400px] bg-[#0a0a0a] rounded-[12px] flex items-center p-1 mx-auto mb-4">
          <div className="p-2">
            <Globe size={20} className="text-gray-500" />
          </div>
          <input
            type="text"
            placeholder="https://example.com"
            value={url}
            onChange={(e) => setUrl(e.target.value)}
            className="flex-grow h-full bg-transparent text-[#e4e4e4] focus:outline-none focus:ring-0 placeholder-gray-600 text-sm"
          />
          <button
            onClick={() => analyzeSite()}
            disabled={loading}
            className="h-[35px] w-[35px] bg-[#e4e4e4] rounded-[8px] flex items-center justify-center flex-shrink-0 transition-opacity disabled:opacity-70 disabled:cursor-not-allowed mr-1"
          >
            {loading ? (
              <div className="bg-black rounded-[6px] w-4 h-4 animate-pulse"></div>
            ) : (
              <ArrowUp size={20} className="text-black" />
            )}
          </button>
        </div>

        {!loading && !result && (
          <div className="flex justify-center items-center gap-3 flex-wrap mb-12">
            <span className="text-sm text-gray-500">Try:</span>
            {proposalUrls.map((pUrl) => (
              <button
                key={pUrl}
                onClick={() => handleProposalClick(pUrl)}
                className="h-[30px] w-auto bg-[#0a0a0a] rounded-[12px] flex items-center px-2 transition-transform hover:scale-105"
              >
                <Globe size={14} className="text-gray-500 mr-2" />
                <p className="text-sm text-gray-300">{pUrl}</p>
                <ArrowUp size={16} className="text-gray-500 ml-2" />
              </button>
            ))}
          </div>
        )}

        {error && <p className="text-red-400 bg-red-900/50 p-3 rounded-lg text-center mb-6">{error}</p>}

        {result && (
          <div className="space-y-12">
            <div>
              <h3 className="text-2xl font-bold text-[#e4e4e4] mb-4">🚀 UI Preview with Enhanced Detection</h3>
              <div className="bg-green-900/20 border border-green-600/30 p-3 rounded-lg mb-4">
                <p className="text-green-300 text-sm">
                  <strong>🎯 Analysis Method:</strong> Basic | <strong>High-confidence animations:</strong>{" "}
                  {result.animationFiles.filter((f) => f.confidence > 80).length}
                  {result.animationFiles.length > 0 && (
                    <span className="block mt-1">
                      📚 Libraries detected:{" "}
                      {[...new Set(result.animationFiles.map((f) => f.library).filter(Boolean))].join(", ")}
                    </span>
                  )}
                </p>
              </div>
              <iframe
                title="UI Preview with Enhanced Animation Detection"
                className="w-full h-96 border border-gray-800/80 rounded-xl bg-white"
                srcDoc={createOptimizedPreview()}
                sandbox="allow-scripts allow-same-origin"
              />
            </div>

            <div>
              <h2 className="text-2xl font-bold text-[#e4e4e4] mb-4">Analysis Results</h2>
              <div className="border-t border-gray-800/50">
                <ResultItem label="Title" value={result.title} />
                <ResultItem label="Description" value={result.description} />
                <ResultItem label="Technologies" value={result.techGuesses.join(", ") || "None detected"} />
                <ResultItem label="Animation Files" value={result.animationFiles.length} />
                <ResultItem
                  label="High-confidence Animations"
                  value={result.animationFiles.filter((f) => f.confidence > 80).length}
                />
                <ResultItem label="CDN Libraries" value={result.requiredCdnUrls.length} />
                <ResultItem label="Internal Links" value={result.internalLinks} />
                <ResultItem label="External Links" value={result.externalLinks} />
                <ResultItem label="Stylesheets" value={result.stylesheets} />
                <ResultItem label="Open Graph Tags" value={result.openGraphTags} />
              </div>
            </div>

            <div>
              <h2 className="text-2xl font-bold text-[#e4e4e4] mb-4">🎭 Enhanced Animation Detection</h2>
              <div className="bg-gray-900/50 p-4 rounded-lg space-y-3">
                {result.animationFiles.length > 0 ? (
                  <>
                    {result.animationFiles
                      .sort((a, b) => b.confidence - a.confidence)
                      .map((file, index) => (
                        <div key={index} className="flex items-center justify-between p-3 bg-gray-800/50 rounded-lg">
                          <div>
                            <p className="text-sm font-medium text-green-400">{file.library || "Generic Animation"}</p>
                            <p className="text-xs text-gray-400 truncate max-w-md">{file.url}</p>
                            <div className="flex items-center gap-2 mt-1">
                              <div
                                className={`px-2 py-1 rounded text-xs ${
                                  file.confidence > 80
                                    ? "bg-green-900/50 text-green-300"
                                    : file.confidence > 60
                                      ? "bg-yellow-900/50 text-yellow-300"
                                      : "bg-red-900/50 text-red-300"
                                }`}
                              >
                                {file.confidence}% confidence
                              </div>
                              {file.library && (
                                <p className="text-xs text-blue-300">
                                  🔗 CDN: {getLibraryCDN(file.library).length} files
                                </p>
                              )}
                            </div>
                          </div>
                          <span
                            className={`px-2 py-1 rounded text-xs ${
                              file.type === "css" ? "bg-blue-900/50 text-blue-300" : "bg-yellow-900/50 text-yellow-300"
                            }`}
                          >
                            {file.type.toUpperCase()}
                          </span>
                        </div>
                      ))}
                  </>
                ) : (
                  <p className="text-gray-400 text-center py-4">No high-confidence animation files detected.</p>
                )}

                <div className="text-xs text-gray-400 space-y-1 mt-4 pt-4 border-t border-gray-700">
                  <p>
                    🎯 <strong>Enhanced Detection:</strong> Blacklist filtering, confidence scoring, and better pattern
                    matching.
                  </p>
                  <p>
                    📊 <strong>Confidence Levels:</strong> Green (80%+), Yellow (60-80%), Red (&lt;60%)
                  </p>
                </div>
              </div>
            </div>

            {result && analyzeAnimationQuality() && (
              <div>
                <h2 className="text-2xl font-bold text-[#e4e4e4] mb-4">Enhanced Animation Quality Analysis</h2>
                <div className="bg-gray-900/50 p-4 rounded-lg">
                  {(() => {
                    const stats = analyzeAnimationQuality()!
                    return (
                      <div className="grid grid-cols-2 md:grid-cols-4 gap-4 mb-4">
                        <div className="text-center">
                          <div className="text-2xl font-bold text-green-400">{stats.highConfidence}</div>
                          <div className="text-xs text-gray-400">High Confidence</div>
                        </div>
                        <div className="text-center">
                          <div className="text-2xl font-bold text-yellow-400">{stats.mediumConfidence}</div>
                          <div className="text-xs text-gray-400">Medium Confidence</div>
                        </div>
                        <div className="text-center">
                          <div className="text-2xl font-bold text-blue-400">{stats.totalFiles}</div>
                          <div className="text-xs text-gray-400">Total Files</div>
                        </div>
                        <div className="text-center">
                          <div className="text-2xl font-bold text-purple-400">{stats.libraries.length}</div>
                          <div className="text-xs text-gray-400">Libraries</div>
                        </div>
                      </div>
                    )
                  })()}

                  <div className="space-y-2">
                    <div className="flex justify-between items-center">
                      <span className="text-sm text-gray-300">Estimated Completeness</span>
                      <span className="text-sm font-medium text-green-400">
                        {analyzeAnimationQuality()!.estimatedCompleteness}%
                      </span>
                    </div>
                    <div className="w-full bg-gray-700 rounded-full h-2">
                      <div
                        className="bg-gradient-to-r from-red-500 via-yellow-500 to-green-500 h-2 rounded-full transition-all duration-500"
                        style={{ width: `${analyzeAnimationQuality()!.estimatedCompleteness}%` }}
                      ></div>
                    </div>
                  </div>

                  {analyzeAnimationQuality()!.libraries.length > 0 && (
                    <div className="mt-4">
                      <p className="text-sm text-gray-300 mb-2">Detected Libraries with CDN:</p>
                      <div className="flex flex-wrap gap-2">
                        {analyzeAnimationQuality()!.libraries.map((lib, index) => (
                          <span key={index} className="px-2 py-1 bg-green-900/50 text-green-300 rounded text-xs">
                            🔗 {lib}
                          </span>
                        ))}
                      </div>
                    </div>
                  )}
                </div>
              </div>
            )}

            <div>
              <h3 className="text-2xl font-bold text-[#e4e4e4] mb-4">Assets</h3>
              <div className="space-y-3">
                <div className="h-[35px] flex-grow flex items-center border border-[#222] rounded-[14px]">
                  <button
                    onClick={() => createDownloadLink(result.fullHTML, "analyzed-site.html", "text/html")}
                    className="bg-black h-full flex-grow flex items-center justify-center px-4 border-r border-[#222] hover:bg-[#111] transition-colors rounded-l-[13px] text-sm font-medium"
                  >
                    Download HTML
                  </button>
                  <button
                    onClick={() => copyToClipboard(result.fullHTML, "html")}
                    className="h-[35px] w-12 flex items-center justify-center bg-transparent hover:bg-[#111] transition-colors rounded-r-[13px]"
                  >
                    <Copy size={15} className="text-gray-400" />
                  </button>
                </div>

                <div className="h-[35px] flex-grow flex items-center border border-[#222] rounded-[14px]">
                  <button
                    onClick={() => createDownloadLink(result.fullCSS, "analyzed-site.css", "text/css")}
                    className="bg-black h-full flex-grow flex items-center justify-center px-4 border-r border-[#222] hover:bg-[#111] transition-colors rounded-l-[13px] text-sm font-medium"
                  >
                    Download CSS
                  </button>
                  <button
                    onClick={() => copyToClipboard(result.fullCSS, "css")}
                    className="h-[35px] w-12 flex items-center justify-center bg-transparent hover:bg-[#111] transition-colors rounded-r-[13px]"
                  >
                    <Copy size={15} className="text-gray-400" />
                  </button>
                </div>

                <div className="h-[35px] flex-grow flex items-center border border-[#222] rounded-[14px]">
                  <button
                    onClick={() => createDownloadLink(result.fullJS, "analyzed-site.js", "text/javascript")}
                    className="bg-black h-full flex-grow flex items-center justify-center px-4 border-r border-[#222] hover:bg-[#111] transition-colors rounded-l-[13px] text-sm font-medium"
                  >
                    Download JS (with animations)
                  </button>
                  <button
                    onClick={() => copyToClipboard(result.fullJS, "js")}
                    className="h-[35px] w-12 flex items-center justify-center bg-transparent hover:bg-[#111] transition-colors rounded-r-[13px]"
                  >
                    <Copy size={15} className="text-gray-400" />
                  </button>
                </div>

                <div className="h-[35px] flex-grow flex items-center border border-green-600/30 rounded-[14px] bg-green-900/20">
                  <button
                    onClick={() =>
                      createDownloadLink(
                        createOptimizedPreview(),
                        "complete-site-with-enhanced-animations.html",
                        "text/html",
                      )
                    }
                    className="bg-black h-full flex-grow flex items-center justify-center px-4 border-r border-green-600/30 hover:bg-[#111] transition-colors rounded-l-[13px] text-sm font-medium text-green-300"
                  >
                    🚀 Download Site with Enhanced Animations
                  </button>
                  <button
                    onClick={() => copyToClipboard(createOptimizedPreview(), "complete")}
                    className="h-[35px] w-12 flex items-center justify-center bg-transparent hover:bg-[#111] transition-colors rounded-r-["
                  >
                    <Copy size={15} className="text-green-400" />
                  </button>
                </div>

                <div className="h-[35px] flex-grow flex items-center border border-[#222] rounded-[14px]">
                  <button
                    onClick={() => {
                      const animations = extractAnimationsOnly()
                      createDownloadLink(animations.css, "animations-only.css", "text/css")
                    }}
                    className="bg-black h-full flex-grow flex items-center justify-center px-4 border-r border-[#222] hover:bg-[#111] transition-colors rounded-l-[13px] text-sm font-medium"
                  >
                    Download Animations CSS
                  </button>
                  <button
                    onClick={() => {
                      const animations = extractAnimationsOnly()
                      copyToClipboard(animations.css, "animations-css")
                    }}
                    className="h-[35px] w-12 flex items-center justify-center bg-transparent hover:bg-[#111] transition-colors rounded-r-[13px]"
                  >
                    <Copy size={15} className="text-gray-400" />
                  </button>
                </div>

                <div className="h-[35px] flex-grow flex items-center border border-[#222] rounded-[14px]">
                  <button
                    onClick={() => {
                      const animations = extractAnimationsOnly()
                      createDownloadLink(animations.js, "animations-only.js", "text/javascript")
                    }}
                    className="bg-black h-full flex-grow flex items-center justify-center px-4 border-r border-[#222] hover:bg-[#111] transition-colors rounded-l-[13px] text-sm font-medium"
                  >
                    Download Animations JS
                  </button>
                  <button
                    onClick={() => {
                      const animations = extractAnimationsOnly()
                      copyToClipboard(animations.js, "animations-js")
                    }}
                    className="h-[35px] w-12 flex items-center justify-center bg-transparent hover:bg-[#111] transition-colors rounded-r-[13px]"
                  >
                    <Copy size={15} className="text-gray-400" />
                  </button>
                </div>
              </div>
            </div>
          </div>
        )}
      </div>

      {result && (
        <div className="fixed bottom-0 left-0 right-0 p-4 flex justify-center backdrop-blur-sm">
          <div className="flex items-center gap-3">
            <div className="h-[35px] w-[250px] flex items-center border border-[#222] rounded-[14px] bg-black shadow-lg">
              <button
                onClick={handleCopyPrompt}
                className="h-full flex-grow flex items-center justify-center px-4 border-r border-[#222] hover:bg-[#111] transition-colors rounded-l-[13px] text-sm font-medium"
              >
                Copy AI Prompt
              </button>
              <button
                onClick={handleDownloadPrompt}
                className="h-[35px] w-12 flex items-center justify-center bg-transparent hover:bg-[#111] transition-colors rounded-r-[13px]"
              >
                <Download size={15} className="text-gray-400" />
              </button>
            </div>
            {copyStatus?.id === "prompt" && (
              <span className="text-sm text-green-400 bg-black/50 p-2 rounded-md">{copyStatus.message}</span>
            )}
          </div>
        </div>
      )}
    </div>
  )
}
