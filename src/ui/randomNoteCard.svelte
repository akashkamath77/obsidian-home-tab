<script lang="ts">
    import { TFile, View } from 'obsidian';
    import { onMount } from 'svelte';

    export let view: View;

    type NoteGlimpse = {
        file: TFile;
        excerpt: string;
    };

    const SWIPE_THRESHOLD_PX = 80;

    let activeCardEl: HTMLDivElement;
    let notePool: TFile[] = [];
    let currentNote: NoteGlimpse | null = null;
    let nextNote: NoteGlimpse | null = null;

    let startX = 0;
    let deltaX = 0;
    let isDragging = false;
    let isAnimating = false;

    const getRandomNote = (): TFile | null => {
        if (notePool.length === 0) return null;
        return notePool[Math.floor(Math.random() * notePool.length)];
    };

    const getExcerpt = async (file: TFile): Promise<string> => {
        try {
            const content = await view.app.vault.cachedRead(file);
            const cleanLine = content
                .split('\n')
                .map((line) => line.trim())
                .find((line) => line.length > 0);
            return cleanLine ? cleanLine.slice(0, 140) : 'No text preview available.';
        }
        catch {
            return 'Preview unavailable.';
        }
    };

    const buildGlimpse = async (file: TFile | null): Promise<NoteGlimpse | null> => {
        if (!file) return null;
        return {
            file,
            excerpt: await getExcerpt(file)
        };
    };

    const loadRandomPair = async (): Promise<void> => {
        notePool = view.app.vault.getMarkdownFiles();

        const first = getRandomNote();
        if (!first) {
            currentNote = null;
            nextNote = null;
            return;
        }

        const second = getRandomNote();
        currentNote = await buildGlimpse(first);
        nextNote = await buildGlimpse(second);
    };

    const pullForwardNextNote = async (): Promise<void> => {
        if (!nextNote) return;

        currentNote = nextNote;
        nextNote = await buildGlimpse(getRandomNote());
    };

    const resetCardPosition = (): void => {
        deltaX = 0;
        isDragging = false;
    };

    const animateCardOutAndAdvance = async (): Promise<void> => {
        if (!activeCardEl || isAnimating) return;

        isAnimating = true;
        const direction = deltaX >= 0 ? 1 : -1;

        activeCardEl.style.transition = 'transform 180ms ease, opacity 180ms ease';
        activeCardEl.style.transform = `translateX(${direction * 120}%) rotate(${direction * 10}deg)`;
        activeCardEl.style.opacity = '0';

        await new Promise((resolve) => setTimeout(resolve, 180));

        await pullForwardNextNote();

        activeCardEl.style.transition = 'none';
        activeCardEl.style.transform = 'translateX(0) rotate(0deg)';
        activeCardEl.style.opacity = '1';

        resetCardPosition();
        isAnimating = false;
    };

    const onPointerDown = (event: PointerEvent): void => {
        if (isAnimating || !currentNote) return;
        startX = event.clientX;
        isDragging = true;
        activeCardEl?.setPointerCapture(event.pointerId);
    };

    const onPointerMove = (event: PointerEvent): void => {
        if (!isDragging || isAnimating) return;
        deltaX = event.clientX - startX;
    };

    const onPointerUp = async (): Promise<void> => {
        if (!isDragging || isAnimating) return;

        if (Math.abs(deltaX) > SWIPE_THRESHOLD_PX) {
            await animateCardOutAndAdvance();
        }
        else {
            resetCardPosition();
        }
    };

    const openCurrentNote = async (): Promise<void> => {
        if (!currentNote || isDragging || isAnimating) return;
        await view.leaf.openFile(currentNote.file);
    };

    const cardStyle = (): string => {
        if (!isDragging) return '';

        const rotate = deltaX / 25;
        return `transform: translateX(${deltaX}px) rotate(${rotate}deg);`;
    };

    onMount(async () => {
        await loadRandomPair();

        view.registerEvent(view.app.vault.on('create', async (file) => {
            if (file instanceof TFile) await loadRandomPair();
        }));

        view.registerEvent(view.app.vault.on('delete', async () => {
            await loadRandomPair();
        }));
    });
</script>

{#if currentNote}
    <section class="random-note-wrapper" aria-label="Random note preview">
        {#if nextNote}
            <article class="random-note-card random-note-card--back" aria-hidden="true">
                <p class="random-note-card__label">Coming up</p>
                <h3>{nextNote.file.basename}</h3>
            </article>
        {/if}

        <article
            class="random-note-card random-note-card--front"
            bind:this={activeCardEl}
            style={cardStyle()}
            on:pointerdown={onPointerDown}
            on:pointermove={onPointerMove}
            on:pointerup={onPointerUp}
            on:pointercancel={onPointerUp}
            on:click={openCurrentNote}>
            <p class="random-note-card__label">Random note</p>
            <h3>{currentNote.file.basename}</h3>
            <p class="random-note-card__path">{currentNote.file.path}</p>
            <p class="random-note-card__excerpt">{currentNote.excerpt}</p>
            <p class="random-note-card__hint">Swipe left/right for another</p>
        </article>
    </section>
{/if}

<style>
    .random-note-wrapper {
        position: relative;
        width: min(90%, 580px);
        margin: 22px auto 8px;
        min-height: 180px;
    }

    .random-note-card {
        position: absolute;
        inset: 0;
        border: 1px solid var(--background-modifier-border);
        background: linear-gradient(150deg, var(--background-secondary), var(--background-primary));
        border-radius: 14px;
        padding: 16px 18px;
        box-shadow:
            0 20px 30px -20px rgba(0, 0, 0, 0.45),
            0 8px 18px -12px rgba(0, 0, 0, 0.35);
    }

    .random-note-card--back {
        transform: translateY(10px) scale(0.975);
        opacity: 0.5;
        filter: saturate(0.8);
        pointer-events: none;
    }

    .random-note-card--front {
        z-index: 2;
        touch-action: pan-y;
        cursor: grab;
        transition: transform 110ms ease;
    }

    .random-note-card--front:active {
        cursor: grabbing;
    }

    .random-note-card__label {
        margin: 0 0 8px;
        font-size: 0.75rem;
        text-transform: uppercase;
        letter-spacing: 0.08em;
        color: var(--text-muted);
    }

    .random-note-card h3 {
        margin: 0;
        font-size: clamp(1rem, 2.4vw, 1.25rem);
        line-height: 1.3;
    }

    .random-note-card__path {
        margin: 6px 0 0;
        font-size: 0.78rem;
        color: var(--text-muted);
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
    }

    .random-note-card__excerpt {
        margin: 14px 0 0;
        line-height: 1.45;
        color: var(--text-normal);
        display: -webkit-box;
        -webkit-line-clamp: 3;
        -webkit-box-orient: vertical;
        overflow: hidden;
    }

    .random-note-card__hint {
        margin: 14px 0 0;
        font-size: 0.78rem;
        color: var(--text-faint);
    }

    @media (max-width: 680px) {
        .random-note-wrapper {
            width: 94%;
            min-height: 194px;
            margin-top: 16px;
        }

        .random-note-card {
            border-radius: 12px;
            padding: 14px;
        }
    }
</style>
