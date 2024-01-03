<script>
    import { onMount } from "svelte";
    import Authentication from "../resources/authentication.js";
    import ProjectApi from "../resources/projectapi.js";
    import censor from "../resources/basiccensorship.js";
    import VRHandler from "../vr";
    const ProjectClient = new ProjectApi();

    // Static values
    import LINK from "../resources/urls.js";

    // Components
    import NavigationBar from "$lib/NavigationBar/NavigationBar.svelte";
    import NavigationMargin from "$lib/NavigationBar/NavMargin.svelte";
    import Button from "$lib/Button/Button.svelte";
    import ContentCategory from "$lib/ContentCategory/Component.svelte";
    import LoadingSpinner from "$lib/LoadingSpinner/Spinner.svelte";
    import UserDisplay from "$lib/UserDisplay/Display.svelte";
    import Project from "$lib/Project/Project.svelte";
    import Alert from "$lib/Alert/Alert.svelte";
    import StatusAlert from "$lib/Alert/StatusAlert.svelte";
    // translations
    import LocalizedText from "$lib/LocalizedText/Node.svelte";
    import TranslationHandler from "../resources/translations.js";
    import Language from "../resources/language.js";

    // Icons
    import PenguinConfusedSVG from "../icons/Penguin/confused.svelte";

    let loggedIn = null;
    let langDecided = false;
    let currentLang = "en";

    let ghcommits = [];
    let myFeed = [];
    let updates = [];
    let feedIsEmpty = false;
    let ghcommitsFailed = false;
    let ghcommitsLoaded = false;
    let projectsLoaded = false;
    let projectsFailed = false;

    let projects = {
        today: [],
        featured: [],
    };

    let thingyActive = false;

    const getAndUpdateMyFeed = async () => {
        const feed = await ProjectClient.getMyFeed();
        if (feed.length <= 0) {
            feedIsEmpty = true;
        }
        myFeed = feed;
    };
    const getFeedText = (type, author, content) => {
        switch (type) {
            case "follow":
                return TranslationHandler.text(
                    "feed.following",
                    currentLang
                ).replace("$1", author);
            case "upload":
                return TranslationHandler.text("feed.uploaded", currentLang)
                    .replace("$1", author)
                    .replace("$2", content.name);
            case "remixed":
                return TranslationHandler.text("feed.remixed", currentLang)
                    .replace("$1", author)
                    .replace("$2", content.name);
            case "posted":
                return TranslationHandler.text(
                    "feed.posted",
                    currentLang
                ).replace("$1", author);
        }
    };
    const getFeedUrl = (type, author, content) => {
        switch (type) {
            case "upload":
            case "remixed":
                return `https://studio.penguinmod.com/#${content.id}`;
            case "posted":
                return `/profile?user=${author}&post=${content.id}`;
            default:
                return `/profile?user=${author}`;
        }
    };

    onMount(async () => {
        const projectId = Number(location.hash.replace("#", ""));
        if (!isNaN(projectId) && projectId != 0) {
            location.href = `https://studio.penguinmod.com/#${projectId}`;
            return;
        }

        fetch(`${LINK.basicApi}commits`)
            .then((res) => {
                res.json()
                    .then((commits) => {
                        ghcommits = commits;
                        ghcommitsLoaded = true;
                    })
                    .catch(() => {
                        ghcommitsFailed = true;
                    });
            })
            .catch(() => {
                ghcommitsFailed = true;
            });
        fetch(`${LINK.basicApi}updates`).then((res) => {
            res.json().then((updatess) => {
                // currently multiple updates are not supported
                updates = [updatess];
            });
        });

        ProjectApi.getMaxProjects(15, false, true).then((projs) => {
            projects.today = projs;
        });
        ProjectApi.getMaxProjects(15, true, false)
            .then((projs) => {
                projects.featured = projs;
                projectsLoaded = true;
            })
            .catch(() => {
                projectsFailed = true;
            });
    });

    // login code below
    let loggedInUsername = "";
    onMount(async () => {
        const privateCode = localStorage.getItem("PV");
        if (!privateCode) {
            loggedIn = false;
            return;
        }
        Authentication.usernameFromCode(privateCode)
            .then(({ username }) => {
                if (username) {
                    loggedInUsername = username;
                    ProjectClient.setUsername(username);
                    ProjectClient.setPrivateCode(privateCode);
                    loggedIn = true;
                    getAndUpdateMyFeed();
                    return;
                }
                loggedIn = false;
            })
            .catch(() => {
                loggedIn = false;
            });
    });

    Authentication.onLogout(() => {
        loggedIn = false;
        myFeed = [];
    });
    Authentication.onAuthentication((privateCode) => {
        loggedIn = null;
        Authentication.usernameFromCode(privateCode)
            .then(({ username }) => {
                if (username) {
                    loggedInUsername = username;
                    ProjectClient.setUsername(username);
                    ProjectClient.setPrivateCode(privateCode);
                    loggedIn = true;
                    getAndUpdateMyFeed();
                    return;
                }
                loggedIn = false;
            })
            .catch(() => {
                loggedIn = false;
            });
    });

    onMount(() => {
        Language.forceUpdate();
    });
    Language.onChange((lang) => {
        currentLang = lang;
        langDecided = true;
    });

    let selectedFrontTabSelected = "new";

    // VR stuff
    let isLiveTests = false;
    let vrIsSupported = null;
    /**
     * @type {VRHandler}
     */
    let vrSession;
    onMount(async () => {
        const urlParams = new URLSearchParams(location.search);
        if (urlParams.has("livetests")) {
            isLiveTests = true;
        }

        if (!isLiveTests) return;
        vrIsSupported = await VRHandler.isSupported();
        if (!vrIsSupported) return;
        vrSession = new VRHandler();
        vrSession.initialize();
    });
    const vr_openSession = () => {
        vrSession.start();
    };
</script>

<svelte:head>
    <title>PenguinMod - You Won!</title>
    <meta name="title" content="PenguinMod - You Won!" />
    <meta property="og:title" content="PenguinMod - You Won!" />
    <meta property="twitter:title" content="PenguinMod - You Won!">
    <meta name="description" content="The area where featured projects and community stuff & info is shown.">
    <meta property="twitter:description" content="The area where featured projects and community stuff & info is shown.">
    <meta property="og:url" content="https://penguinmod.com/">
    <meta property="twitter:url" content="https://penguinmod.com/">
</svelte:head>

<NavigationBar />

<div class="main">
    <NavigationMargin />

    <Alert
        onlyShowID={"donatee:_2"}
        text={"PenguinMod Premium has just been added! We are giving it away to a lucky few users!"}
        textBreakup={true}
        textColor={"white"}
        hasImage={true}
        imgSrc={"/happy.svg"}
        imgAlt={":D"}
        hasButton={true}
        buttonText={"Join Giveaway"}
        buttonHref={"https://www.youtube.com/watch?v=dQw4w9WgXcQ"}
    />
    <!-- TODO: re-add this, but only have it appear for new users after they login on a date before the alert -->
    <!-- <Alert
        onlyShowID={"privacee:_1"}
        text={"Our privacy policy has been updated."}
        textBreakup={true}
        textColor={"white"}
        backColor={"#009900"}
        hasImage={false}
        hasButton={true}
        buttonText={"View"}
        buttonHref={"https://studio.penguinmod.com/privacy.html"}
    /> -->
    <StatusAlert />

    {#if loggedIn === false}
        <div class="section-info">
            <div style="margin-left: 8rem;">
                <h1>
                    Congrats! You won PenguinMod Premium!
                </h1>
                <p>Only <b>37</b> subscriptions available! Redeem quickly!</p>
                <Button
                    label="<img src='/tryit.svg' width='32px' style='margin-right:8px'></img>"
                    highlighted="true"
                    link={"https://www.youtube.com/watch?v=dQw4w9WgXcQ"}
                >
                    {#if !thingyActive}
                        Redeem Premium!
                    {:else}
                        Redeem Premium!
                    {/if}
                </Button>
            </div>

            {#if !thingyActive}
                <video
                    width="426.666667"
                    height="240"
                    autoplay="true"
                    muted="true"
                    loop="true"
                    class="example-video"
                >
                    <source src="/example.mp4" type="video/mp4" />
                    <track kind="captions" />
                </video>
            {:else}
                <iframe
                    src="/eao.html"
                    title="The Thingy"
                    width="426.666667"
                    height="240"
                    frameborder="0"
                    class="example-video"
                />
            {/if}
        </div>

        {#if langDecided && currentLang != "en" && loggedIn === false}
            <div class="section-language-warning">
                <img
                    src="/warning.png"
                    draggable="false"
                    style="height: 24px; margin-right: 6px"
                    alt="Warning"
                />
                <p>
                    <LocalizedText
                        text="PenguinMod is made by English-speaking developers. Expect minor issues and sorry for any translation errors."
                        key="translation.warning"
                        lang={currentLang}
                    />
                </p>
            </div>
        {/if}
    {/if}
        
    {#if isLiveTests && vrIsSupported}
        <button
            class="vr-test-button"
            on:click={vr_openSession}
        >
            Enter VR
        </button>
    {/if}

    {#if langDecided && currentLang != "en" && loggedIn !== false}
        <div class="section-language-warning">
            <img
                src="/warning.png"
                draggable="false"
                style="height: 24px; margin-right: 6px"
                alt="Warning"
            />
            <p>
                <LocalizedText
                    text="PenguinMod is made by English-speaking developers. Expect minor issues and sorry for any translation errors."
                    key="translation.warning"
                    lang={currentLang}
                />
            </p>
        </div>
    {/if}
    {#if loggedIn}
        <div class="section-category-toggles">
            <div class="category-toggle-section" />
            <div class="category-toggle-section">
                <button
                    class="section-toggle-button"
                    data-active={selectedFrontTabSelected === "new"}
                    on:click={() => {
                        selectedFrontTabSelected = "new";
                    }}
                >
                    <LocalizedText
                        text="What's new?"
                        key="home.sections.whatsnew"
                        lang={currentLang}
                    />
                </button>
                <button
                    class="section-toggle-button"
                    data-active={selectedFrontTabSelected === "feed"}
                    on:click={() => {
                        selectedFrontTabSelected = "feed";
                    }}
                >
                    <LocalizedText
                        text="My Feed"
                        key="home.sections.feed"
                        lang={currentLang}
                    />
                </button>
                <button
                    class="section-toggle-button"
                    data-active={selectedFrontTabSelected === "commit"}
                    on:click={() => {
                        selectedFrontTabSelected = "commit";
                    }}
                >
                    <LocalizedText
                        text="Recent commits"
                        key="home.sections.githubcommits"
                        lang={currentLang}
                    />
                </button>
            </div>
        </div>
    {/if}

    <div class="section-projects">
        <center><h1>What does PenguinMod Premium get you?</h1></center>
        <center><h3>Verified Profile Badge</h3>
            <p>Ever wanted to look like a dork- I mean cool person? Verified checkmark allows you to tell everyone you pay $8 per month for PenguinMod!</p>
        </center>
        <center><h3>Rainbow Theme</h3>
            <p>With PenguinMod Premium, you get access to our cutting edge JavaScript code to enable Rainbow theme on PenguinMod! Please don't use ChatGPT to do it yourself.</p>
        </center>
        <center><h3>Remove Extension Limit</h3>
            <p>Since we want money, we are now limiting the amount of extensions you can have in a project to <b>3</b>. With Premium, you can add unlimited!</p>
        </center>
        <center><h3>Remove Block Limit</h3>
            <p>Since we want money, we are now limiting the amount of blocks you can have in a project to <b>20</b>. With Premium, you can add unlimited!</p>
        </center>
        <center><h3>Remove Project Limit</h3>
            <p>Since we want money, we are now limiting the amount of projects you can have on your account to <b>5</b>. With Premium, you can have unlimited!</p>
        </center>
        <center><h3>Remove Screen Time Limit</h3>
            <p>Since we want money, we are now limiting the amount of time you can spend on our website to <b>30 minutes per day</b>. With Premium, you can have unlimited time!</p>
        </center>
        <center><h3>Logging in</h3>
            <p>Since we want money, we are now preventing anyone from logging in or signing up. With Premium, you can log in or signup!</p>
        </center>
        <center><h3>Viewing Projects</h3>
            <p>Since we want money, we are now preventing anyone from viewing projects. With Premium, you can view projects!</p>
        </center>
    </div>

    <div class="footer">
        <p>
            <!-- {#if !thingyActive} -->
                <LocalizedText
                    text="PenguinMod is not affiliated with Scratch, TurboWarp, the Scratch Team, or the Scratch Foundation."
                    key="home.footer.notaffiliated"
                    dontlink={true}
                    lang={currentLang}
                />
            <!-- todo: find a better place to put this that isn't, the legal text -->
            <!-- {:else}
                EEAAOO EEAAOOEEAAOOEEAAOOEEAAOOEEAAOOEEAAOO EEAAOO
                EEAAOOEEAAOOEEAAOO EEAAOO
            {/if} -->
        </p>
        <div class="footer-list">
            <div class="footer-section">
                <p>
                    <LocalizedText
                        text="Website"
                        key="home.footer.sections.website"
                        lang={currentLang}
                    />
                </p>
                <a href={LINK.editor}>
                    <LocalizedText
                        text="Editor"
                        key="home.footer.sections.website.editor"
                        lang={currentLang}
                    />
                </a>
                <a href={LINK.credits}>
                    <LocalizedText
                        text="Credits"
                        key="home.footer.sections.website.credits"
                        lang={currentLang}
                    />
                </a>
                <a href={LINK.github}>
                    <LocalizedText
                        text="Source"
                        key="home.footer.sections.website.source"
                        lang={currentLang}
                    />
                </a>
                <a href={LINK.packager}>
                    <LocalizedText
                        text="Packager"
                        key="home.footer.sections.website.packager"
                        lang={currentLang}
                    />
                </a>
            </div>
            <div class="footer-section">
                <p>
                    <LocalizedText
                        text="Community"
                        key="home.footer.sections.community"
                        lang={currentLang}
                    />
                </p>
                <a target="_blank" href={LINK.discord}>Discord</a>
                <a target="_blank" href={LINK.wiki}>
                    <LocalizedText
                        text="Wiki"
                        key="home.footer.sections.community.wiki"
                        lang={currentLang}
                    />
                </a>
            </div>
            <div class="footer-section">
                <p>
                    <LocalizedText
                        text="Info"
                        key="home.footer.sections.info"
                        lang={currentLang}
                    />
                </p>
                <a href={LINK.terms}>
                    <LocalizedText
                        text="Terms of Service"
                        key="home.footer.sections.info.terms"
                        lang={currentLang}
                    />
                </a>
                <a href={LINK.privacy}>
                    <LocalizedText
                        text="Privacy Policy"
                        key="home.footer.sections.info.privacy"
                        lang={currentLang}
                    />
                </a>
                <a target="_blank" href={"/guidelines/uploading"}>
                    <LocalizedText
                        text="Uploading Guidelines"
                        key="home.footer.sections.info.guidelines"
                        lang={currentLang}
                    />
                </a>
            </div>
            <div class="footer-section">
                <p>
                    <LocalizedText
                        text="Donate"
                        key="home.footer.sections.donate"
                        lang={currentLang}
                    />
                </p>
                <a href={"/donate"}>PenguinMod</a>
                <a target="_blank" href={LINK.donate.turbowarp}>TurboWarp</a>
                <a target="_blank" href={LINK.donate.scratch}>Scratch</a>
            </div>
        </div>
    </div>
</div>

<style>
    * {
        font-family: "Helvetica Neue", Helvetica, Arial, sans-serif;
    }

    .footer {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        width: 100%;
        padding: 24px 0px 48px;
        border-top: rgba(0, 0, 0, 0.3) 1px solid;
        background: #00c3ff15;
        font-weight: bold;
        margin-top: 4px;
        /* border-top-left-radius: 20%; */
        /* border-top-right-radius: 20%; */
    }
    .footer a {
        color: dodgerblue;
        font-weight: normal;
        margin: 2px 0px;
    }
    .footer a:active {
        color: rgb(15, 77, 139);
    }
    .footer-list {
        display: flex;
        align-items: flex-start;
        justify-content: center;
    }
    .footer-section {
        display: flex;
        flex-direction: column;
        margin: 0px 32px;
        font-size: 14px;
    }
    :global(body.dark-mode) .footer {
        /* border-top: rgba(255, 255, 255, 0.1) 1px solid; */
        border-top: rgba(255, 255, 255, 0.3) 1px solid;
        /* background: transparent; */
        /* background: #0059ff15; */
        /* border-top-left-radius: 20%; */
        /* border-top-right-radius: 20%; */
    }

    .main {
        position: absolute;
        left: 0px;
        top: 0px;
        width: 100%;
        min-width: 1000px;
    }
    :global(body.dark-mode) .main {
        color: white;
    }

    .section-info {
        background: #00c3ffad;
        height: 24rem;
        color: white;
        display: flex;
        flex-direction: row;
        align-items: center;
        justify-content: space-between;
        width: 100%;
        margin: 0;
    }
    :global(html[dir="rtl"]) .section-info {
        justify-content: space-around;
    }
    .section-links {
        background: #00c3ff28;
        color: white;
        display: flex;
        flex-direction: row;
        align-items: center;
        justify-content: center;
        width: 100%;
        padding: 0.5rem 0;
        margin: 0px;
    }

    .section-categories {
        display: flex;
        flex-direction: row;
        align-items: center;
        justify-content: center;
        width: 100%;
        margin: 0px;
    }
    .section-category-toggles {
        display: flex;
        flex-direction: row;
        align-items: center;
        justify-content: center;
        width: 100%;
        margin: 0px;
    }
    .category-toggle-section {
        display: flex;
        flex-direction: row;
        align-items: center;
        justify-content: center;
        width: 30%;
        margin: 4px 10px;
    }
    .section-toggle-button {
        border-radius: 1024px;
        padding: 4px 10px;
        background: #008cff;
        font-weight: bold;
        font-size: 1em;
        border: 0;
        margin: 0 4px;
        color: white;
        cursor: pointer;
    }
    .section-toggle-button[data-active="true"] {
        background: #003bdd;
    }

    .profile-picture {
        width: 72px;
        height: 72px;
        border-radius: 4px;
    }
    .welcome-back-card {
        width: 30%;
        height: 312px;
        margin: 10px;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
    }
    .welcome-back-row {
        display: flex;
        flex-direction: row;
        align-items: center;
    }
    .welcome-back-button {
        display: flex;
        flex-direction: column;
        align-items: center;
        background: transparent;
        border: 0;
        cursor: pointer;
    }
    :global(body.dark-mode) .welcome-back-button {
        color: white;
    }
    .welcome-back-no-underline {
        text-decoration: none;
    }
    .welcome-back-icon-container {
        border: 1px solid rgba(0, 0, 0, 0.25);
        background: transparent;
        border-radius: 8px;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        width: 72px;
        height: 69px;
        margin-bottom: 8px;
    }
    :global(body.dark-mode) .welcome-back-icon-container {
        border: 1px solid rgba(255, 255, 255, 0.5);
    }
    .welcome-back-button:active .welcome-back-icon-container {
        background: rgba(0, 0, 0, 0.2);
    }
    .welcome-back-row >
    a:first-child .welcome-back-icon-container {
        border-top-left-radius: 36px;
        border-bottom-left-radius: 36px;
        padding-left: 8px;
    }
    .welcome-back-row >
    a:last-child .welcome-back-icon-container {
        border-top-right-radius: 36px;
        border-bottom-right-radius: 36px;
        padding-right: 8px;
    }
    :global(body.dark-mode)
        .welcome-back-button:active
        .welcome-back-icon-container {
        background: rgba(255, 255, 255, 0.2);
    }
    .welcome-back-icon-container img {
        width: 32px;
        height: 32px;
        filter: brightness(0.2);
    }
    :global(body.dark-mode) .welcome-back-icon-container img {
        filter: brightness(1);
    }

    .section-projects {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        width: 100%;
        margin: 0px;
    }
    .section-language-warning {
        background: #ffd00073;
        color: black;
        display: flex;
        flex-direction: row;
        align-items: center;
        justify-content: center;
        width: 100%;
        font-size: 18px;
        font-weight: bold;
        margin: 0px;
        text-align: center;
    }
    .section-language-warning > img {
        filter: brightness(0);
    }
    :global(body.dark-mode) .section-language-warning {
        color: white;
    }
    :global(body.dark-mode) .section-language-warning > img {
        filter: brightness(1);
    }

    .example-video {
        border-radius: 6px;
        outline-style: solid;
        outline-width: 6px;
        outline-color: rgba(255, 255, 255, 0.35);
        margin-right: 8rem;
    }

    .category-content {
        display: flex;
        align-items: center;
        flex-direction: column;
    }

    .update-image {
        width: 100%;
        height: 100%;
    }
    .update-image-wrapper {
        background: transparent;
        cursor: pointer;
        margin-top: 4px;
        width: 100%;
        height: 100%;
        padding: 0;
        margin: 0;
        border: 0;
    }

    .project-list {
        display: flex;
        flex-direction: row;
    }

    /* test styles, remove later */
    .vr-test-button {
        padding: 20px;
        margin: 4px;
        font-size: larger;
    }
</style>
