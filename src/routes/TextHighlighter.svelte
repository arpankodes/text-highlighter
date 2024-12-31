<script>
    import { createEventDispatcher, onDestroy, onMount } from "svelte";

    let highlightButtons;

    const dispatch = createEventDispatcher();

    onMount(() => {
        document.addEventListener('mouseup', showHighlightButtons);
        document.addEventListener('mousedown', hideHighlightButtons);
        document.addEventListener('scroll', hideHighlightButtons);
        initRemoveHighlight();
    });

    onDestroy(() => {
        document.removeEventListener('click', hideHighlightButtons);
        document.removeEventListener('scroll', hideHighlightButtons);
    });

    function initRemoveHighlight() {
        const removeButton = document.getElementById('remove-highlight-button');
        if (!removeButton) return;
        removeButton.addEventListener('click', () => {
            if (highlightButtons.range) {
                removeHighlight(highlightButtons.range);
                highlightButtons.style.display = 'none';
            }
        });
    }

    function applyHighlight(range, color) {
        const isTextNode = node => node.nodeType === Node.TEXT_NODE;
        
        function expandToWordBoundaries(range) {
            const startNode = range.startContainer;
            const endNode = range.endContainer;
            
            if (isTextNode(startNode)) {
                let startOffset = range.startOffset;
                const text = startNode.textContent;
                while (startOffset > 0 && !/\s/.test(text[startOffset - 1])) {
                    startOffset--;
                }
                range.setStart(startNode, startOffset);
            }
            
            if (isTextNode(endNode)) {
                let endOffset = range.endOffset;
                const text = endNode.textContent;
                while (endOffset < text.length && !/\s/.test(text[endOffset])) {
                    endOffset++;
                }
                range.setEnd(endNode, endOffset);
            }
        }

        try {
            expandToWordBoundaries(range);
            const findParagraph = (node) => {
                if (!node) return null;
                const elementNode = node.nodeType === Node.TEXT_NODE ? node.parentElement : node;
                return elementNode?.closest('p') || elementNode?.parentElement?.closest('p');
            };
            
            const startParagraph = findParagraph(range.startContainer);
            const endParagraph = findParagraph(range.endContainer);
            
            if (startParagraph && endParagraph && startParagraph !== endParagraph) {
                const firstRange = range.cloneRange();
                firstRange.setEnd(startParagraph, startParagraph.childNodes.length);
                
                const lastRange = range.cloneRange();
                lastRange.setStart(endParagraph, 0);
                
                let currentNode = startParagraph.nextElementSibling;
                const middleParagraphs = [];
                while (currentNode && currentNode !== endParagraph) {
                    if (currentNode.matches('p')) {
                        middleParagraphs.push(currentNode);
                    }
                    currentNode = currentNode.nextElementSibling;
                }
                
                const firstSpan = document.createElement('span');
                firstSpan.classList.add(`highlight-${color}`);
                const firstContents = firstRange.extractContents();
                firstSpan.appendChild(firstContents);
                firstRange.insertNode(firstSpan);
                
                middleParagraphs.forEach(p => {
                    const span = document.createElement('span');
                    span.classList.add(`highlight-${color}`);
                    span.appendChild(p.cloneNode(true));
                    p.replaceWith(span);
                });
                
                const lastSpan = document.createElement('span');
                lastSpan.classList.add(`highlight-${color}`);
                const lastContents = lastRange.extractContents();
                lastSpan.appendChild(lastContents);
                lastRange.insertNode(lastSpan);
            } else {
                const span = document.createElement('span');
                span.classList.add(`highlight-${color}`);
                const contents = range.extractContents();
                span.appendChild(contents);
                range.insertNode(span);
            }
            
            if (startParagraph) {
                startParagraph.normalize();
            }
            if (endParagraph && endParagraph !== startParagraph) {
                endParagraph.normalize();
            }
            dispatch('highlight', { range, color });
            window.getSelection()?.removeAllRanges();
        } catch (error) {
            console.error('Error applying highlight:', error);
            window.getSelection()?.removeAllRanges();
        }
    }

    function showHighlightButtons(e) {
        const selection = window.getSelection();
        if (!selection?.isCollapsed) {
            const range = selection?.getRangeAt(0);
            const rect = range?.getBoundingClientRect();
            if (!rect) return;
            if (highlightButtons && (!e || !highlightButtons.contains(e.target))) {
                highlightButtons.style.top = `${window.scrollY + rect.top - 40}px`;
                highlightButtons.style.left = `${window.scrollX + rect.left}px`;
                highlightButtons.style.display = 'flex';
                highlightButtons.range = range;
            }
        }
    }

    function hideHighlightButtons(e) {
        if (highlightButtons && (!e || !highlightButtons.contains(e.target))) {
            highlightButtons.style.display = 'none';
            highlightButtons.range = null;
        }
    }

    function handleHighlightClick(color) {
        if (highlightButtons.range) {
            applyHighlight(highlightButtons.range, color);
            hideHighlightButtons();
        }
    }

    function removeHighlight(range) {
        const findParagraph = (node) => {
            if (!node) return null;
            const elementNode = node.nodeType === Node.TEXT_NODE ? node.parentElement : node;
            return elementNode?.closest('p') || elementNode?.parentElement?.closest('p');
        };

        const startParagraph = findParagraph(range.startContainer);
        const endParagraph = findParagraph(range.endContainer);

        const processHighlightSpans = (container) => {
            const highlightSpans = container.querySelectorAll('span.highlight-yellow, span.highlight-lightblue');
            highlightSpans.forEach(span => {
                const spanRange = document.createRange();
                spanRange.selectNodeContents(span);

                if (range.compareBoundaryPoints(Range.END_TO_START, spanRange) < 0 &&
                    range.compareBoundaryPoints(Range.START_TO_END, spanRange) > 0) {
                    const intersectionRange = spanRange.cloneRange();
                    if (range.startContainer === span) {
                        intersectionRange.setStart(range.startContainer, range.startOffset);
                    }
                    if (range.endContainer === span) {
                        intersectionRange.setEnd(range.endContainer, range.endOffset);
                    }

                    const extractedContent = intersectionRange.extractContents();
                    const parent = span.parentNode;
                    while (span.firstChild) {
                        parent.insertBefore(span.firstChild, span);
                    }
                    parent.removeChild(span);
                }
            });
        };

        if (startParagraph && endParagraph) {
            let currentNode = startParagraph;
            while (currentNode) {
                if (currentNode.matches('p') || currentNode.matches('div')) {
                    processHighlightSpans(currentNode);
                }
                if (currentNode === endParagraph) break;
                currentNode = currentNode.nextElementSibling;
            }
        } else {
            const container = range.commonAncestorContainer;
            processHighlightSpans(container);
        }

        dispatch('removeHighlight', { range });
        if (startParagraph) startParagraph.normalize();
        if (endParagraph && endParagraph !== startParagraph) endParagraph.normalize();
        window.getSelection()?.removeAllRanges();
    }

</script>

<div bind:this={highlightButtons} class="highlight-buttons">
    <button class="yellow-btn" on:click={() => handleHighlightClick('yellow')}></button>
    <button class="blue-btn" on:click={() => handleHighlightClick('lightblue')}></button>
    <button id="remove-highlight-button" class="remove-highlight"><b>&#8722;</b></button>
</div>

<style>

    .highlight-buttons {
        flex-direction: row;
        gap: 6px;
        position: absolute;
        z-index: 1000;
        display: none;
        gap: 6px;
        padding: 6px;
        background: rgba(255, 255, 255, 0.95);
        backdrop-filter: blur(4px);
        border-radius: 50px;
        box-shadow: 
            0 4px 6px -1px rgba(0, 0, 0, 0.1),
            0 2px 4px -1px rgba(0, 0, 0, 0.06);
        transform: translateY(-10px);
        animation: fadeIn 0.2s ease-out;
        width: fit-content;
    }

    @keyframes fadeIn {
        from {
            opacity: 0;
            transform: translateY(0px);
        }
        to {
            opacity: 1;
            transform: translateY(-10px);
        }
    }

    .highlight-buttons button {
        width: 26px;
        height: 26px;
        border-radius: 50%;
        border: none;
        cursor: pointer;
        transition: all 0.2s ease;
        position: relative;
        display: flex;
        align-items: center;
        justify-content: center;
        padding: 0;
        margin: 0;
    }

    .highlight-buttons button:hover {
        transform: scale(1.1);
        box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
    }

    .highlight-buttons button:active {
        transform: scale(0.95);
    }

    .yellow-btn {
        background: #FFD700;
        box-shadow: 0 2px 4px rgba(255, 215, 0, 0.3);
    }

    .blue-btn {
        background: #87CEEB;
        box-shadow: 0 2px 4px rgba(135, 206, 235, 0.3);
    }

    .remove-highlight {
        background: white;
        box-shadow: 0 2px 4px rgba(255, 68, 68, 0.2);
    }

    :global(.highlight-yellow) {
        background-color: yellow;
    }

    :global(.highlight-lightblue) {
        background-color: lightblue;
    }

    :global(.highlight-yellow),
    :global(.highlight-lightblue) {
        display: inline;
    }

</style>