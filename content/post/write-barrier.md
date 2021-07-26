---
title: "Write Barrier"
date: 2021-07-22T06:27:34-04:00
draft: true
---

{{<html>}}
<style>
.hidden {
    display: none;
}
</style>

<div id="al-svg-1">
    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg">
        <style>
            .al-code {
                font-family: monospace;
            }
            .active-slide > rect {
                stroke: #4477AA;
            }
            .active-slide > text {
                fill: #4477AA;
            }
        </style>
        <rect width=300 height=200 fill=white stroke="grey" stroke-width="5px" />

        <g transform="translate(25 10)">
            <text class="al-code" font-size=15 y="1em" dy="0em">foo = "first"</text>
            <text class="al-code" font-size=15 y="1em" dy="1em">puts("foo")</text>
            <text class="al-code" font-size=15 y="1em" dy="2em">foo = {}</text>
            <text class="al-code" font-size=15 y="1em" dy="3em" >foo[0] = "second"</text>
            <text class="al-code" font-size=15 y="1em" dy="4em" >foo[1] = "third"</text>
            <text class="al-code" font-size=15 y="1em" dy="5em">GC.start</text>
        </g>

        <rect fill="#EE6677" width="6" height="6" rx="1" x="13" y="0.7em">
            <animate id=root1 attributeName=y dur="1.0s" values="0.7em;4.3em" fill=freeze calcMode=spline keySplines="0.23 0.73 0.7 1" />
        </rect>

        <g>
            <rect width=50 height=20 x=180 y=20 fill=white stroke=black />
            <text class="al-code" x=205 y=32 font-size=11 textLength=40 text-anchor=middle>"first"</text>
            <animate attributeName=opacity dur=1.7s values="0;1" begin=root1.begin />
        </g>

        <g opacity=0>
            <rect width=50 height=20 x=230 y=60 fill=white stroke=black />
            <text class="al-code" x=255 y=73 font-size=11 text-anchor=middle>{}</text>
            <animate attributeName=opacity dur=1.7s values="0;1" begin="root1.begin+0.3s" fill=freeze />
        </g>


        <g opacity=0>
            <rect width=50 height=20 x=200 y=100 fill=white stroke=black />
            <text class="al-code" x=225 y=112 font-size=10 textLength=45 text-anchor=middle>"second"</text>
            <line x1=225 y1=100 x2=255 y2=80 stroke=black />
            <animate attributeName=opacity dur=1.7s values="0;1" begin="root1.begin+0.6s" fill=freeze />
        </g>

        <g opacity=0>
            <rect width=50 height=20 x=238 y=130 fill=white stroke=black />
            <text class="al-code" x=263 y=143 font-size=10 textLength=40 text-anchor=middle>"third"</text>
            <line x1=263 y1=130 x2=255 y2=80 stroke=black />
            <animate attributeName=opacity dur=1.7s values="0;1" begin="root1.begin+0.9" fill=freeze />
        </g>


        <text class="al-code" font-size="10" text-anchor="middle" x="150" y="153">Code allocates objects</text>

        <a xlink:href="#stop-the-world-1">
            <rect y="170" x="121" width="16" height="16" fill=white stroke=#4477AA></rect>
            <text class="al-code" y="182" x="129" font-size=10 text-anchor=middle textLength=10 fill=#4477AA>1</text>
        </a>
        <a xlink:href="#stop-the-world-2">
            <rect y="170" x="142" width="16" height="16" stroke=black fill=white></rect>
            <text class="al-code" y="182" x="150" font-size=10 text-anchor=middle textLength=10 >2</text>
        </a>
        <a xlink:href="#stop-the-world-3">
            <rect y="170" x="163" width="16" height="16" stroke=black fill=none></rect>
            <text class="al-code" y="182" x="171" font-size=10 text-anchor=middle textLength=10>3</text>

    </svg>

    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg" class=hidden>
        <style>
            .al-code {
                font-family: monospace;
            }
        </style>
        <rect width=300 height=200 fill=white stroke="grey" stroke-width="5px" />

        <g transform="translate(25 10)">
            <text class="al-code" font-size=15 y="1em" dy="0em">foo = "first"</text>
            <text class="al-code" font-size=15 y="1em" dy="1em">puts("foo")</text>
            <text class="al-code" font-size=15 y="1em" dy="2em">foo = {}</text>
            <text class="al-code" font-size=15 y="1em" dy="3em" >foo[0] = "second"</text>
            <text class="al-code" font-size=15 y="1em" dy="4em" >foo[1] = "third"</text>
            <text class="al-code" font-size=15 y="1em" dy="5em">GC.start</text>
        </g>

        <rect fill="#EE6677" width="6" height="6" rx="1" x="13" y="1.3em">
            <animate id=root2 attributeName=y dur="1.0s" values="4.3em;5.1em" fill=freeze calcMode=spline keySplines="0.23 0.73 0.7 1" />
        </rect>

        <g>
            <rect width=50 height=20 x=180 y=20 fill=white stroke=black />
            <text class="al-code" x=205 y=32 font-size=11 textLength=40 text-anchor=middle>"first"</text>
        </g>

        <text textAnchor=middle x=13 y=120 font-size=10 text-decoration=underline>Legend</text>
        <rect fill=white width="6" height="6" stroke=black rx="1" x="13" y="125" />
        <text class=al-code x=22 y=130 font-size=6>Not marked</text>
        <rect fill=#CCBB44 width="6" height="6" stroke=black rx="1" x="13" y=134 />
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propogate, then become marked.</text>
        <rect fill=#228833 width="6" height="6" stroke=black rx="1" x="13" y=143 />
        <text class=al-code x=22 y=148 font-size=6>Marked</text>

        <g>
            <rect width=50 height=20 x=230 y=60 fill=#CCBB44 stroke=black>
                <animate begin="root2.begin" attributeName=fill values="#CCBB44; #CCBB44; #CCBB44; #228833" fill=freeze dur="3s" keyTimes="0;0.33;0.8;1" />
            </rect>
            <text class="al-code" x=255 y=73 font-size=11 text-anchor=middle>{}</text>
        </g>


        <rect width=50 height=20 x=200 y=100 fill=white stroke=black>
            <animate attributeName=fill dur=1s values="white;#CCBB44" fill=freeze begin="root2.begin+1" />
            <animate attributeName=fill dur=1s values="#CCBB44;#228833" fill=freeze begin="root2.begin+3.5" />
        </rect>
        <text class="al-code" x=225 y=112 font-size=10 textLength=45 text-anchor=middle>"second"</text>
        <line x1=225 y1=100 x2=255 y2=80 stroke=black>
            <animate begin="root2.begin" attributeName=stroke values="black; #CCBB44; #CCBB44; black" fill=freeze dur="3s" keyTimes="0;0.33;0.66;1" />
            <animate begin="root2.begin" attributeName=stroke-width values="1; 3; 3; 1" fill=freeze dur="3s" keyTimes="0;0.33;0.66;1" />
        </line>

        <rect width=50 height=20 x=238 y=130 fill=white stroke=black>
            <animate begin="root2.begin+1" attributeName=fill dur=1s values="white;#CCBB44" fill=freeze />
            <animate begin="root2.begin+3.5" attributeName=fill dur=1s values="#CCBB44;#228833" fill=freeze />
        </rect>
        <text class="al-code" x=263 y=143 font-size=10 textLength=40 text-anchor=middle>"third"</text>
        <line x1=263 y1=130 x2=255 y2=80 stroke=black>
            <animate begin="root2.begin" attributeName=stroke values="black; #CCBB44; #CCBB44; black" fill=freeze dur="3s" keyTimes="0;0.33;0.66;1" />
            <animate begin="root2.begin" attributeName=stroke-width values="1; 3; 3; 1" fill=freeze dur="3s" keyTimes="0;0.33;0.66;1" />
        </line>

        <text class="al-code" font-size="10" text-anchor="middle" x="150" y="153">GC runs</text>
        <text class="al-code" font-size="10" text-anchor="middle" x="150" y="153" dy=1em>Starts knowing `foo` needs marking</text>

        <a xlink:href="#stop-the-world-1">
            <rect y="170" x="121" width="16" height="16" fill=white stroke=black></rect>
            <text class="al-code" y="182" x="129" font-size=10 text-anchor=middle textLength=10 fill=black>1</text>
        </a>
        <a xlink:href="#stop-the-world-2">
            <rect y="170" x="142" width="16" height="16" stroke=#4477AA fill=white></rect>
            <text class="al-code" y="182" x="150" font-size=10 text-anchor=middle textLength=10 fill=#4477AA>2</text>
        </a>
        <a xlink:href="#stop-the-world-3">
            <rect y="170" x="163" width="16" height="16" stroke=black fill=none></rect>
            <text class="al-code" y="182" x="171" font-size=10 text-anchor=middle textLength=10>3</text>
        </a>

    </svg>

    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg" class=hidden>
        <style>
            .al-code {
                font-family: monospace;
            }
        </style>
        <rect width=300 height=200 fill=white stroke="grey" stroke-width="5px" />

        <g transform="translate(25 10)">
            <text class="al-code" font-size=15 y="1em" dy="0em">foo = "first"</text>
            <text class="al-code" font-size=15 y="1em" dy="1em">puts("foo")</text>
            <text class="al-code" font-size=15 y="1em" dy="2em">foo = {}</text>
            <text class="al-code" font-size=15 y="1em" dy="3em" >foo[0] = "second"</text>
            <text class="al-code" font-size=15 y="1em" dy="4em" >foo[1] = "third"</text>
            <text class="al-code" font-size=15 y="1em" dy="5em">GC.start</text>
        </g>

        <rect fill="#EE6677" width="6" height="6" rx="1" x="13" y="5.1em">
        </rect>

        <g>
            <rect width=50 height=20 x=180 y=20 fill=white stroke=black />
            <text class="al-code" x=205 y=32 font-size=11 textLength=40 text-anchor=middle>"first"</text>
        </g>

        <text textAnchor=middle x=13 y=120 font-size=10 text-decoration=underline>Legend</text>
        <rect fill=white width="6" height="6" stroke=black rx="1" x="13" y="125" />
        <text class=al-code x=22 y=130 font-size=6>Not marked</text>
        <rect fill=#CCBB44 width="6" height="6" stroke=black rx="1" x="13" y=134 />
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propogate, then become marked.</text>
        <rect fill=#228833 width="6" height="6" stroke=black rx="1" x="13" y=143 />
        <text class=al-code x=22 y=148 font-size=6>Marked</text>

        <g>
            <rect width=50 height=20 x=230 y=60 fill=#228833 stroke=black>
            </rect>
            <text class="al-code" x=255 y=73 font-size=11 text-anchor=middle>{}</text>
        </g>


        <rect width=50 height=20 x=200 y=100 fill=#228833 stroke=black>
        </rect>
        <text class="al-code" x=225 y=112 font-size=10 textLength=45 text-anchor=middle>"second"</text>
        <line x1=225 y1=100 x2=255 y2=80 stroke=black>
        </line>

        <rect width=50 height=20 x=238 y=130 fill=#228833 stroke=black>
        </rect>
        <text class="al-code" x=263 y=143 font-size=10 textLength=40 text-anchor=middle>"third"</text>
        <line x1=263 y1=130 x2=255 y2=80 stroke=black>
        </line>

        <text class="al-code" font-size="10" text-anchor="middle" x="150" y="153" dy=1em>Done. "first" can be released</text>

        <a xlink:href="#stop-the-world-1">
            <rect y="170" x="121" width="16" height="16" fill=white stroke=black></rect>
            <text class="al-code" y="182" x="129" font-size=10 text-anchor=middle textLength=10 fill=black>1</text>
        </a>
        <a xlink:href="#stop-the-world-2">
            <rect y="170" x="142" width="16" height="16" stroke=black fill=white></rect>
            <text class="al-code" y="182" x="150" font-size=10 text-anchor=middle textLength=10>2</text>
        </a>
        <a xlink:href="#stop-the-world-3">
            <rect y="170" x="163" width="16" height="16" stroke=#4477AA fill=none></rect>
            <text class="al-code" y="182" x="171" font-size=10 text-anchor=middle fill=#4477AA textLength=10>3</text>
        </a>

    </svg>
</div>

<div id="al-svg-2">
    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg">
        <style>
            .al-code {
                font-family: monospace;
            }
        </style>
        <rect width=300 height=200 fill=white stroke="grey" stroke-width="5px" />

        <g transform="translate(25 10)">
            <text class="al-code" font-size=15 y="1em" dy="0em">foo = "first"</text>
            <text class="al-code" font-size=15 y="1em" dy="1em">foo = {}</text>
            <text class="al-code" font-size=15 y="1em" dy="2em">foo[0] = []</text>
            <text class="al-code" font-size=15 y="1em" dy="3em"># GC step</text>
            <text class="al-code" font-size=15 y="1em" dy="4em">foo[0] << "second"</text>
            <text class="al-code" font-size=15 y="1em" dy="5em"># GC step</text>
        </g>

        <rect fill="#EE6677" width="6" height="6" rx="1" x="13" y="1.3em">
            <animate id=root3 attributeName=y dur="1.0s" values=".8em;2.6em" fill=freeze calcMode=spline keySplines="0.23 0.73 0.7 1" />
        </rect>

        <g>
            <rect width=50 height=20 x=180 y=20 fill=white stroke=black />
            <text class="al-code" x=205 y=32 font-size=11 textLength=40 text-anchor=middle>"first"</text>
            <animate begin=root3.begin attributeName=opacity dur=1.7s values="0;1" />
        </g>

        <text textAnchor=middle x=13 y=120 font-size=10 text-decoration=underline>Legend</text>
        <rect fill=white width="6" height="6" stroke=black rx="1" x="13" y="125" />
        <text class=al-code x=22 y=130 font-size=6>Not marked</text>
        <rect fill=#CCBB44 width="6" height="6" stroke=black rx="1" x="13" y=134 />
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propogate, then become marked.</text>
        <rect fill=#228833 width="6" height="6" stroke=black rx="1" x="13" y=143 />
        <text class=al-code x=22 y=148 font-size=6>Marked</text>

        <g opacity=0>
            <rect width=50 height=20 x=230 y=60 fill=none stroke=black>
            </rect>
            <text class="al-code" x=255 y=73 font-size=11 text-anchor=middle>{}</text>
            <animate attributeName=opacity dur=1.7s values="0;1" begin=root3.begin+0.3 fill=freeze />
        </g>


        <g opacity=0>
            <rect width=50 height=20 x=200 y=95 fill=none stroke=black>
            </rect>
            <text class="al-code" x=225 y=105 dominant-baseline=middle font-size=10 text-anchor=middle>[]</text>
            <line x1=225 y1=95 x2=255 y2=80 stroke=black>
            </line>
            <animate attributeName=opacity dur=1.7s values="0;1" begin="root3.begin+0.7" fill=freeze />
        </g>

        <g opacity=0>
            <rect width=50 height=20 x=238 y=130 fill=none stroke=black>
            </rect>
            <text class="al-code" x=263 y=143 font-size=10 textLength=40 text-anchor=middle>"third"</text>
            <line x1=263 y1=130 x2=225 y2=115 stroke=black>
            </line>
        </g>

        <text class="al-code" font-size="10" text-anchor="middle" x="150" y="153" dy=1em>Code allocates objects</text>

        <svg viewBox="0 0 100 20" width=100 height=20 y=170 x=101>
            <a xlink:href="#incremental-1" class="active-slide">
                <rect y="2" x="5" width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x=13 font-size=10 dominant-baseline=middle text-anchor=middle textLength=10 fill=black>1</text>
            </a>
            <a xlink:href="#incremental-2">
                <rect y=2 x=26 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="34" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>2</text>
            </a>
            <a xlink:href="#incremental-3">
                <rect y=2 x=47 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="55" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>3</text>
            </a>
            <a xlink:href="#incremental-4">
                <rect y=2 x=68 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="76" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>4</text>
            </a>
        </svg>
    </svg>

    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg" class=hidden>
        <style>
            .al-code {
                font-family: monospace;
            }
        </style>
        <rect width=300 height=200 fill=white stroke="grey" stroke-width="5px" />

        <g transform="translate(25 10)">
            <text class="al-code" font-size=15 y="1em" dy="0em">foo = "first"</text>
            <text class="al-code" font-size=15 y="1em" dy="1em">foo = {}</text>
            <text class="al-code" font-size=15 y="1em" dy="2em">foo[0] = []</text>
            <text class="al-code" font-size=15 y="1em" dy="3em"># GC step</text>
            <text class="al-code" font-size=15 y="1em" dy="4em">foo[0] << "second"</text>
            <text class="al-code" font-size=15 y="1em" dy="5em"># GC step</text>
        </g>

        <rect fill="#EE6677" width="6" height="6" rx="1" x="13" y="1.3em">
            <animate id=root4 attributeName=y dur="1.0s" values="2.6em;3.4em" fill=freeze calcMode=spline keySplines="0.23 0.73 0.7 1" />
        </rect>

        <g>
            <rect width=50 height=20 x=180 y=20 fill=white stroke=black />
            <text class="al-code" x=205 y=32 font-size=11 textLength=40 text-anchor=middle>"first"</text>
        </g>

        <text textAnchor=middle x=13 y=120 font-size=10 text-decoration=underline>Legend</text>
        <rect fill=white width="6" height="6" stroke=black rx="1" x="13" y="125" />
        <text class=al-code x=22 y=130 font-size=6>Not marked</text>
        <rect fill=#CCBB44 width="6" height="6" stroke=black rx="1" x="13" y=134 />
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propogate, then become marked.</text>
        <rect fill=#228833 width="6" height="6" stroke=black rx="1" x="13" y=143 />
        <text class=al-code x=22 y=148 font-size=6>Marked</text>

        <g>
            <rect width=50 height=20 x=230 y=60 fill=none stroke=black>
                <animate begin=root4.begin attributeName=fill values="#CCBB44; #CCBB44; #CCBB44; #228833" fill=freeze dur="3s" keyTimes="0;0.33;0.8;1" />
            </rect>
            <text class="al-code" x=255 y=73 font-size=11 text-anchor=middle>{}</text>
        </g>

        <g>
            <rect width=50 height=20 x=200 y=95 fill=none stroke=black>
                <animate attributeName=fill dur=1s values="white;#CCBB44" fill=freeze begin=root4.begin+1 />
            </rect>
            <text class="al-code" x=225 y=105 dominant-baseline=middle font-size=10 text-anchor=middle>[]</text>
            <line x1=225 y1=95 x2=255 y2=80 stroke=black>
                <animate begin=root4.begin attributeName=stroke values="black; #CCBB44; #CCBB44; black" fill=freeze dur="3s" keyTimes="0;0.33;0.66;1" />
                <animate begin=root4.begin attributeName=stroke-width values="1; 3; 3; 1" fill=freeze dur="3s" keyTimes="0;0.33;0.66;1" />
            </line>
        </g>

        <g opacity=0>
            <rect width=50 height=20 x=238 y=130 fill=none stroke=black>
            </rect>
            <text class="al-code" x=263 y=143 font-size=10 textLength=40 text-anchor=middle>"third"</text>
            <line x1=263 y1=130 x2=225 y2=115 stroke=black>
            </line>
        </g>

        <text class="al-code" font-size="10" text-anchor="middle" x="150" y="153" dy=1em>GC runs for a bit and stops halfway</text>

        <svg viewBox="0 0 100 20" width=100 height=20 y=170 x=101>
            <a xlink:href="#incremental-1">
                <rect y="2" x="5" width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x=13 font-size=10 dominant-baseline=middle text-anchor=middle textLength=10 fill=black>1</text>
            </a>
            <a xlink:href="#incremental-2" class="active-slide">
                <rect y=2 x=26 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="34" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>2</text>
            </a>
            <a xlink:href="#incremental-3">
                <rect y=2 x=47 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="55" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>3</text>
            </a>
            <a xlink:href="#incremental-4">
                <rect y=2 x=68 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="76" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>4</text>
            </a>
        </svg>
    </svg>

    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg" class=hidden>
        <style>
            .al-code {
                font-family: monospace;
            }
        </style>
        <rect width=300 height=200 fill=white stroke="grey" stroke-width="5px" />

        <g transform="translate(25 10)">
            <text class="al-code" font-size=15 y="1em" dy="0em">foo = "first"</text>
            <text class="al-code" font-size=15 y="1em" dy="1em">foo = {}</text>
            <text class="al-code" font-size=15 y="1em" dy="2em">foo[0] = []</text>
            <text class="al-code" font-size=15 y="1em" dy="3em"># GC step</text>
            <text class="al-code" font-size=15 y="1em" dy="4em">foo[0] << "second"</text>
            <text class="al-code" font-size=15 y="1em" dy="5em"># GC step</text>
        </g>

        <rect fill="#EE6677" width="6" height="6" rx="1" x="13" y="1.3em">
            <animate id=root5 attributeName=y dur="1.0s" values="3.4em;4.3em" fill=freeze calcMode=spline keySplines="0.23 0.73 0.7 1" />
        </rect>

        <g>
            <rect width=50 height=20 x=180 y=20 fill=white stroke=black />
            <text class="al-code" x=205 y=32 font-size=11 textLength=40 text-anchor=middle>"first"</text>
        </g>

        <text textAnchor=middle x=13 y=120 font-size=10 text-decoration=underline>Legend</text>
        <rect fill=white width="6" height="6" stroke=black rx="1" x="13" y="125" />
        <text class=al-code x=22 y=130 font-size=6>Not marked</text>
        <rect fill=#CCBB44 width="6" height="6" stroke=black rx="1" x="13" y=134 />
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propogate, then become marked.</text>
        <rect fill=#228833 width="6" height="6" stroke=black rx="1" x="13" y=143 />
        <text class=al-code x=22 y=148 font-size=6>Marked</text>

        <g>
            <rect width=50 height=20 x=230 y=60 fill=#228833 stroke=black>
            </rect>
            <text class="al-code" x=255 y=73 font-size=11 text-anchor=middle>{}</text>
        </g>

        <g>
            <rect width=50 height=20 x=200 y=95 fill=#CCBB44 stroke=black>
            </rect>
            <text class="al-code" x=225 y=105 dominant-baseline=middle font-size=10 text-anchor=middle>[]</text>
            <line x1=225 y1=95 x2=255 y2=80 stroke=black>
            </line>
        </g>

        <g opacity=0>
            <rect width=50 height=20 x=238 y=130 fill=none stroke=black>
            </rect>
            <text class="al-code" x=263 y=143 font-size=10 textLength=45 text-anchor=middle>"second"</text>
            <line x1=263 y1=130 x2=225 y2=115 stroke=black>
            </line>
            <animate begin=root5.begin attributeName=opacity dur=1.7s values="0;1" fill=freeze />
        </g>

        <text class="al-code" font-size="10" text-anchor="middle" x="150" y="153" dy=1em>Code resumes</text>

        <svg viewBox="0 0 100 20" width=100 height=20 y=170 x=101>
            <a xlink:href="#incremental-1">
                <rect y="2" x="5" width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x=13 font-size=10 dominant-baseline=middle text-anchor=middle textLength=10 fill=black>1</text>
            </a>
            <a xlink:href="#incremental-2">
                <rect y=2 x=26 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="34" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>2</text>
            </a>
            <a xlink:href="#incremental-3" class="active-slide">
                <rect y=2 x=47 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="55" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>3</text>
            </a>
            <a xlink:href="#incremental-4">
                <rect y=2 x=68 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="76" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>4</text>
            </a>
        </svg>
    </svg>

    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg" class=hidden>
        <style>
            .al-code {
                font-family: monospace;
            }
        </style>
        <rect width=300 height=200 fill=white stroke="grey" stroke-width="5px" />

        <g transform="translate(25 10)">
            <text class="al-code" font-size=15 y="1em" dy="0em">foo = "first"</text>
            <text class="al-code" font-size=15 y="1em" dy="1em">foo = {}</text>
            <text class="al-code" font-size=15 y="1em" dy="2em">foo[0] = []</text>
            <text class="al-code" font-size=15 y="1em" dy="3em"># GC step</text>
            <text class="al-code" font-size=15 y="1em" dy="4em">foo[0] << "second"</text>
            <text class="al-code" font-size=15 y="1em" dy="5em"># GC step</text>
        </g>

        <rect fill="#EE6677" width="6" height="6" rx="1" x="13" y="1.3em">
            <animate id=root6 attributeName=y dur="1.0s" values="4.3em;5.1em" fill=freeze calcMode=spline keySplines="0.23 0.73 0.7 1" />
        </rect>

        <g>
            <rect width=50 height=20 x=180 y=20 fill=white stroke=black />
            <text class="al-code" x=205 y=32 font-size=11 textLength=40 text-anchor=middle>"first"</text>
        </g>

        <text textAnchor=middle x=13 y=120 font-size=10 text-decoration=underline>Legend</text>
        <rect fill=white width="6" height="6" stroke=black rx="1" x="13" y="125" />
        <text class=al-code x=22 y=130 font-size=6>Not marked</text>
        <rect fill=#CCBB44 width="6" height="6" stroke=black rx="1" x="13" y=134 />
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propogate, then become marked.</text>
        <rect fill=#228833 width="6" height="6" stroke=black rx="1" x="13" y=143 />
        <text class=al-code x=22 y=148 font-size=6>Marked</text>

        <g>
            <rect width=50 height=20 x=230 y=60 fill=#228833 stroke=black>
            </rect>
            <text class="al-code" x=255 y=73 font-size=11 text-anchor=middle>{}</text>
        </g>

        <g>
            <rect width=50 height=20 x=200 y=95 fill=#CCBB44 stroke=black>
                <animate begin=root6.begin attributeName=fill values="#CCBB44; #CCBB44; #CCBB44; #228833" fill=freeze dur="3s" keyTimes="0;0.33;0.8;1" />
            </rect>
            <text class="al-code" x=225 y=105 dominant-baseline=middle font-size=10 text-anchor=middle>[]</text>
            <line x1=225 y1=95 x2=255 y2=80 stroke=black>
            </line>
        </g>

        <g>
            <rect width=50 height=20 x=238 y=130 fill=none stroke=black>
                <animate attributeName=fill dur=1s values="white;#CCBB44" fill=freeze begin=root6.begin+1 />
                <animate attributeName=fill dur=1s values="#CCBB44;#228833" fill=freeze begin=root6.begin+3.2 />
            </rect>
            <text class="al-code" x=263 y=143 font-size=10 textLength=45 text-anchor=middle>"second"</text>
            <line x1=263 y1=130 x2=225 y2=115 stroke=black>
                <animate begin=root6.begin attributeName=stroke values="black; #CCBB44; #CCBB44; black" fill=freeze dur="3s" keyTimes="0;0.33;0.66;1" />
                <animate begin=root6.begin attributeName=stroke-width values="1; 3; 3; 1" fill=freeze dur="3s" keyTimes="0;0.33;0.66;1" />
            </line>
        </g>

        <text class="al-code" font-size="10" text-anchor="middle" x="150" y="153" dy=1em>GC resumes and completes</text>

        <svg viewBox="0 0 100 20" width=100 height=20 y=170 x=101>
            <a xlink:href="#incremental-1">
                <rect y="2" x="5" width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x=13 font-size=10 dominant-baseline=middle text-anchor=middle textLength=10 fill=black>1</text>
            </a>
            <a xlink:href="#incremental-2">
                <rect y=2 x=26 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="34" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>2</text>
            </a>
            <a xlink:href="#incremental-3">
                <rect y=2 x=47 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="55" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>3</text>
            </a>
            <a xlink:href="#incremental-4" class="active-slide">
                <rect y=2 x=68 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="76" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>4</text>
            </a>
        </svg>
    </svg>
</div>

<div id="al-svg-3">
    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg">
        <style>
            .al-code {
                font-family: monospace;
            }
        </style>
        <rect width=300 height=200 fill=white stroke="grey" stroke-width="5px" />

        <text class=al-code font-size=6 y=195 x=6 textLength=80>Missing write barrier</text>

        <g transform="translate(25 10)">
            <text class="al-code" font-size=15 y="1em" dy="0em">foo = "first"</text>
            <text class="al-code" font-size=15 y="1em" dy="1em">foo = {}</text>
            <text class="al-code" font-size=15 y="1em" dy="2em">foo[0] = "second"</text>
            <text class="al-code" font-size=15 y="1em" dy="3em"># GC step</text>
            <text class="al-code" font-size=15 y="1em" dy="4em">foo[1] = "third"</text>
            <text class="al-code" font-size=15 y="1em" dy="5em"># GC step</text>
        </g>

        <rect fill="#EE6677" width="6" height="6" rx="1" x="13" y="1.3em">
            <animate id=root7 attributeName=y dur="1.0s" values="0.8em;2.6em" fill=freeze calcMode=spline keySplines="0.23 0.73 0.7 1" />
        </rect>

        <text textAnchor=middle x=13 y=120 font-size=10 text-decoration=underline>Legend</text>
        <rect fill=white width="6" height="6" stroke=black rx="1" x="13" y="125" />
        <text class=al-code x=22 y=130 font-size=6>Not marked</text>
        <rect fill=#CCBB44 width="6" height="6" stroke=black rx="1" x="13" y=134 />
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propogate, then become marked.</text>
        <rect fill=#228833 width="6" height="6" stroke=black rx="1" x="13" y=143 />
        <text class=al-code x=22 y=148 font-size=6>Marked</text>

        <g>
            <rect width=50 height=20 x=190 y=20 fill=white stroke=black />
            <text class="al-code" x=215 y=32 font-size=11 textLength=40 text-anchor=middle>"first"</text>
            <animate begin=root7.begin attributeName=opacity dur=1.7s values="0;1" />
        </g>

        <g opacity=0>
            <rect width=50 height=20 x=230 y=60 fill=white stroke=black />
            <text class="al-code" x=255 y=73 font-size=11 text-anchor=middle>{}</text>
            <animate attributeName=opacity dur=1.7s values="0;1" begin=root7.begin+0.3 fill=freeze />
        </g>


        <g opacity=0>
            <rect width=50 height=20 x=200 y=100 fill=white stroke=black />
            <text class="al-code" x=225 y=112 font-size=10 textLength=45 text-anchor=middle>"second"</text>
            <line x1=225 y1=100 x2=255 y2=80 stroke=black />
            <animate attributeName=opacity dur=1.7s values="0;1" begin=root7.begin+0.6 fill=freeze />
        </g>

        <g opacity=0>
            <rect width=50 height=20 x=238 y=130 fill=white stroke=black />
            <text class="al-code" x=263 y=143 font-size=10 textLength=40 text-anchor=middle>"third"</text>
            <line x1=263 y1=130 x2=255 y2=80 stroke=black />
        </g>


        <text class="al-code" font-size="10" text-anchor="middle" x="150" y="153" dy=1em>Code runs</text>

        <svg viewBox="0 0 100 20" width=100 height=20 y=170 x=101>
            <a xlink:href="#missing-wb-1" class="active-slide">
                <rect y="2" x="5" width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x=13 font-size=10 dominant-baseline=middle text-anchor=middle textLength=10 fill=black>1</text>
            </a>
            <a xlink:href="#missing-wb-2">
                <rect y=2 x=26 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="34" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>2</text>
            </a>
            <a xlink:href="#missing-wb-3">
                <rect y=2 x=47 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="55" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>3</text>
            </a>
            <a xlink:href="#missing-wb-4">
                <rect y=2 x=68 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="76" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>4</text>
            </a>
        </svg>
    </svg>

    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg" class=hidden>
        <style>
            .al-code {
                font-family: monospace;
            }
        </style>
        <rect width=300 height=200 fill=white stroke="grey" stroke-width="5px" />

        <text class=al-code font-size=6 y=195 x=6 textLength=80>Missing write barrier</text>

        <g transform="translate(25 10)">
            <text class="al-code" font-size=15 y="1em" dy="0em">foo = "first"</text>
            <text class="al-code" font-size=15 y="1em" dy="1em">foo = {}</text>
            <text class="al-code" font-size=15 y="1em" dy="2em">foo[0] = "second"</text>
            <text class="al-code" font-size=15 y="1em" dy="3em"># GC step</text>
            <text class="al-code" font-size=15 y="1em" dy="4em">foo[1] = "third"</text>
            <text class="al-code" font-size=15 y="1em" dy="5em"># GC step</text>
        </g>

        <rect fill="#EE6677" width="6" height="6" rx="1" x="13" y="1.3em">
            <animate id=root8 attributeName=y dur="1.0s" values="2.6em;3.4em" fill=freeze calcMode=spline keySplines="0.23 0.73 0.7 1" />
        </rect>

        <text textAnchor=middle x=13 y=120 font-size=10 text-decoration=underline>Legend</text>
        <rect fill=white width="6" height="6" stroke=black rx="1" x="13" y="125" />
        <text class=al-code x=22 y=130 font-size=6>Not marked</text>
        <rect fill=#CCBB44 width="6" height="6" stroke=black rx="1" x="13" y=134 />
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propogate, then become marked.</text>
        <rect fill=#228833 width="6" height="6" stroke=black rx="1" x="13" y=143 />
        <text class=al-code x=22 y=148 font-size=6>Marked</text>

        <g>
            <rect width=50 height=20 x=190 y=20 fill=white stroke=black />
            <text class="al-code" x=215 y=32 font-size=11 textLength=40 text-anchor=middle>"first"</text>
        </g>

        <g>
            <rect width=50 height=20 x=230 y=60 fill=none stroke=black>
                <animate begin=root8.begin attributeName=fill values="#CCBB44; #CCBB44; #CCBB44; #228833" fill=freeze dur="3s" keyTimes="0;0.33;0.8;1" />
            </rect>
            <text class="al-code" x=255 y=73 font-size=11 text-anchor=middle>{}</text>
        </g>

        <g>
            <rect width=50 height=20 x=200 y=100 fill=white stroke=black>
                <animate attributeName=fill dur=1s values="white;#CCBB44" fill=freeze begin=root8.begin+1 />
                <animate attributeName=fill dur=1s values="#CCBB44;#228833" fill=freeze begin=root8.begin+3.1 />
            </rect>
            <text class="al-code" x=225 y=112 font-size=10 textLength=45 text-anchor=middle>"second"</text>
            <line x1=225 y1=100 x2=255 y2=80 stroke=black>
                <animate begin=root8.begin attributeName=stroke values="black; #CCBB44; #CCBB44; black" fill=freeze dur="3s" keyTimes="0;0.33;0.66;1" />
                <animate begin=root8.begin attributeName=stroke-width values="1; 3; 3; 1" fill=freeze dur="3s" keyTimes="0;0.33;0.66;1" />
            </line>
        </g>


        <g opacity=0>
            <rect width=50 height=20 x=238 y=130 fill=white stroke=black />
            <text class="al-code" x=263 y=143 font-size=10 textLength=40 text-anchor=middle>"third"</text>
            <line x1=263 y1=130 x2=255 y2=80 stroke=black />
        </g>


        <text class="al-code" font-size="10" text-anchor="middle" x="150" y="153" dy=1em>GC runs and finishes marking {}</text>

        <svg viewBox="0 0 100 20" width=100 height=20 y=170 x=101>
            <a xlink:href="#incremental-1">
                <rect y="2" x="5" width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x=13 font-size=10 dominant-baseline=middle text-anchor=middle textLength=10 fill=black>1</text>
            </a>
            <a xlink:href="#incremental-2" class="active-slide">
                <rect y=2 x=26 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="34" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>2</text>
            </a>
            <a xlink:href="#incremental-3">
                <rect y=2 x=47 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="55" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>3</text>
            </a>
            <a xlink:href="#incremental-4">
                <rect y=2 x=68 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="76" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>4</text>
            </a>
        </svg>
    </svg>

    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg" class=hidden>
        <style>
            .al-code {
                font-family: monospace;
            }
        </style>
        <rect width=300 height=200 fill=white stroke="grey" stroke-width="5px" />

        <text class=al-code font-size=6 y=195 x=6 textLength=80>Missing write barrier</text>

        <g transform="translate(25 10)">
            <text class="al-code" font-size=15 y="1em" dy="0em">foo = "first"</text>
            <text class="al-code" font-size=15 y="1em" dy="1em">foo = {}</text>
            <text class="al-code" font-size=15 y="1em" dy="2em">foo[0] = "second"</text>
            <text class="al-code" font-size=15 y="1em" dy="3em"># GC step</text>
            <text class="al-code" font-size=15 y="1em" dy="4em">foo[1] = "third"</text>
            <text class="al-code" font-size=15 y="1em" dy="5em"># GC step</text>
        </g>

        <rect fill="#EE6677" width="6" height="6" rx="1" x="13" y="1.3em">
            <animate id=root9 attributeName=y dur="1.0s" values="3.4em;4.3em" fill=freeze calcMode=spline keySplines="0.23 0.73 0.7 1" />
        </rect>

        <text textAnchor=middle x=13 y=120 font-size=10 text-decoration=underline>Legend</text>
        <rect fill=white width="6" height="6" stroke=black rx="1" x="13" y="125" />
        <text class=al-code x=22 y=130 font-size=6>Not marked</text>
        <rect fill=#CCBB44 width="6" height="6" stroke=black rx="1" x="13" y=134 />
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propogate, then become marked.</text>
        <rect fill=#228833 width="6" height="6" stroke=black rx="1" x="13" y=143 />
        <text class=al-code x=22 y=148 font-size=6>Marked</text>

        <g>
            <rect width=50 height=20 x=190 y=20 fill=white stroke=black />
            <text class="al-code" x=215 y=32 font-size=11 textLength=40 text-anchor=middle>"first"</text>
        </g>

        <g>
            <rect width=50 height=20 x=230 y=60 fill=#228833 stroke=black>
            </rect>
            <text class="al-code" x=255 y=73 font-size=11 text-anchor=middle>{}</text>
        </g>

        <g>
            <rect width=50 height=20 x=200 y=100 fill=#228833 stroke=black>
            </rect>
            <text class="al-code" x=225 y=112 font-size=10 textLength=45 text-anchor=middle>"second"</text>
            <line x1=225 y1=100 x2=255 y2=80 stroke=black>
            </line>
        </g>


        <g opacity=0>
            <rect width=50 height=20 x=238 y=130 fill=white stroke=black />
            <text class="al-code" x=263 y=143 font-size=10 textLength=40 text-anchor=middle>"third"</text>
            <line x1=263 y1=130 x2=255 y2=80 stroke=black />
            <animate begin=root9.begin attributeName=opacity dur=1.7s values="0;1" fill=freeze />
        </g>


        <text class="al-code" font-size="10" text-anchor="middle" x="150" y="153" dy=1em>Code adds new reference</text>

        <svg viewBox="0 0 100 20" width=100 height=20 y=170 x=101>
            <a xlink:href="#incremental-1">
                <rect y="2" x="5" width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x=13 font-size=10 dominant-baseline=middle text-anchor=middle textLength=10 fill=black>1</text>
            </a>
            <a xlink:href="#incremental-2">
                <rect y=2 x=26 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="34" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>2</text>
            </a>
            <a xlink:href="#incremental-3" class="active-slide">
                <rect y=2 x=47 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="55" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>3</text>
            </a>
            <a xlink:href="#incremental-4">
                <rect y=2 x=68 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="76" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>4</text>
            </a>
        </svg>
    </svg>

    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg" class=hidden>
        <style>
            .al-code {
                font-family: monospace;
            }
        </style>
        <rect width=300 height=200 fill=white stroke="grey" stroke-width="5px" />

        <text class=al-code font-size=6 y=195 x=6 textLength=80>Missing write barrier</text>

        <g transform="translate(25 10)">
            <text class="al-code" font-size=15 y="1em" dy="0em">foo = "first"</text>
            <text class="al-code" font-size=15 y="1em" dy="1em">foo = {}</text>
            <text class="al-code" font-size=15 y="1em" dy="2em">foo[0] = "second"</text>
            <text class="al-code" font-size=15 y="1em" dy="3em"># GC step</text>
            <text class="al-code" font-size=15 y="1em" dy="4em">foo[1] = "third"</text>
            <text class="al-code" font-size=15 y="1em" dy="5em"># GC step</text>
        </g>

        <rect fill="#EE6677" width="6" height="6" rx="1" x="13" y="1.3em">
            <animate id=root10 attributeName=y dur="1.0s" values="4.3em;5.0em" fill=freeze calcMode=spline keySplines="0.23 0.73 0.7 1" />
        </rect>

        <text textAnchor=middle x=13 y=120 font-size=10 text-decoration=underline>Legend</text>
        <rect fill=white width="6" height="6" stroke=black rx="1" x="13" y="125" />
        <text class=al-code x=22 y=130 font-size=6>Not marked</text>
        <rect fill=#CCBB44 width="6" height="6" stroke=black rx="1" x="13" y=134 />
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propogate, then become marked.</text>
        <rect fill=#228833 width="6" height="6" stroke=black rx="1" x="13" y=143 />
        <text class=al-code x=22 y=148 font-size=6>Marked</text>

        <g>
            <rect width=50 height=20 x=190 y=20 fill=white stroke=black />
            <text class="al-code" x=215 y=32 font-size=11 textLength=40 text-anchor=middle>"first"</text>
        </g>

        <g>
            <rect width=50 height=20 x=230 y=60 fill=#228833 stroke=black>
            </rect>
            <text class="al-code" x=255 y=73 font-size=11 text-anchor=middle>{}</text>
        </g>

        <g>
            <rect width=50 height=20 x=200 y=100 fill=#228833 stroke=black>
            </rect>
            <text class="al-code" x=225 y=112 font-size=10 textLength=45 text-anchor=middle>"second"</text>
            <line x1=225 y1=100 x2=255 y2=80 stroke=black>
            </line>
        </g>


        <g>
            <rect width=50 height=20 x=238 y=130 fill=white stroke=black />
            <text class="al-code" x=263 y=143 font-size=10 textLength=40 text-anchor=middle>"third"</text>
            <line x1=263 y1=130 x2=255 y2=80 stroke=black />
        </g>


        <text class="al-code" font-size="10" text-anchor="middle" x="150" y="153" dy=1em>"third" incorrectly treated as garbage</text>

        <svg viewBox="0 0 100 20" width=100 height=20 y=170 x=101>
            <a xlink:href="#incremental-1">
                <rect y="2" x="5" width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x=13 font-size=10 dominant-baseline=middle text-anchor=middle textLength=10 fill=black>1</text>
            </a>
            <a xlink:href="#incremental-2">
                <rect y=2 x=26 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="34" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>2</text>
            </a>
            <a xlink:href="#incremental-3">
                <rect y=2 x=47 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="55" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>3</text>
            </a>
            <a xlink:href="#incremental-4" class="active-slide">
                <rect y=2 x=68 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="76" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>4</text>
            </a>
        </svg>
    </svg>
</div>

<div id="al-svg-4">
    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg">
        <style>
            .al-code {
                font-family: monospace;
            }
        </style>
        <rect width=300 height=200 fill=white stroke="grey" stroke-width="5px" />

        <text class=al-code font-size=6 y=195 x=6 textLength=65>With write barrier</text>

        <g transform="translate(25 10)">
            <text class="al-code" font-size=15 y="1em" dy="0em">foo = "first"</text>
            <text class="al-code" font-size=15 y="1em" dy="1em">foo = {}</text>
            <text class="al-code" font-size=15 y="1em" dy="2em">foo[0] = "second"</text>
            <text class="al-code" font-size=15 y="1em" dy="3em"># GC step</text>
            <text class="al-code" font-size=15 y="1em" dy="4em">foo[1] = "third"</text>
            <text class="al-code" font-size=15 y="1em" dy="5em"># GC step</text>
        </g>

        <rect fill="#EE6677" width="6" height="6" rx="1" x="13" y="1.3em">
            <animate id=root11 attributeName=y dur="1.0s" values="0.8em;2.6em" fill=freeze calcMode=spline keySplines="0.23 0.73 0.7 1" />
        </rect>

        <text textAnchor=middle x=13 y=120 font-size=10 text-decoration=underline>Legend</text>
        <rect fill=white width="6" height="6" stroke=black rx="1" x="13" y="125" />
        <text class=al-code x=22 y=130 font-size=6>Not marked</text>
        <rect fill=#CCBB44 width="6" height="6" stroke=black rx="1" x="13" y=134 />
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propogate, then become marked.</text>
        <rect fill=#228833 width="6" height="6" stroke=black rx="1" x="13" y=143 />
        <text class=al-code x=22 y=148 font-size=6>Marked</text>

        <g>
            <rect width=50 height=20 x=190 y=20 fill=white stroke=black />
            <text class="al-code" x=215 y=32 font-size=11 textLength=40 text-anchor=middle>"first"</text>
            <animate begin=root11.begin attributeName=opacity dur=1.7s values="0;1" />
        </g>

        <g opacity=0>
            <rect width=50 height=20 x=230 y=60 fill=white stroke=black />
            <text class="al-code" x=255 y=73 font-size=11 text-anchor=middle>{}</text>
            <animate attributeName=opacity dur=1.7s values="0;1" begin=root11.begin+0.3 fill=freeze />
        </g>


        <g opacity=0>
            <rect width=50 height=20 x=200 y=100 fill=white stroke=black />
            <text class="al-code" x=225 y=112 font-size=10 textLength=45 text-anchor=middle>"second"</text>
            <line x1=225 y1=100 x2=255 y2=80 stroke=black />
            <animate attributeName=opacity dur=1.7s values="0;1" begin=root11.begin+0.6 fill=freeze />
        </g>

        <g opacity=0>
            <rect width=50 height=20 x=238 y=130 fill=white stroke=black />
            <text class="al-code" x=263 y=143 font-size=10 textLength=40 text-anchor=middle>"third"</text>
            <line x1=263 y1=130 x2=255 y2=80 stroke=black />
        </g>


        <text class="al-code" font-size="10" text-anchor="middle" x="150" y="153" dy=1em>Code runs</text>

        <svg viewBox="0 0 100 20" width=100 height=20 y=170 x=101>
            <a xlink:href="#write-barrier-1" class="active-slide">
                <rect y="2" x="5" width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x=13 font-size=10 dominant-baseline=middle text-anchor=middle textLength=10 fill=black>1</text>
            </a>
            <a xlink:href="#write-barrier-2">
                <rect y=2 x=26 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="34" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>2</text>
            </a>
            <a xlink:href="#write-barrier-3">
                <rect y=2 x=47 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="55" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>3</text>
            </a>
            <a xlink:href="#write-barrier-4">
                <rect y=2 x=68 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="76" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>4</text>
            </a>
        </svg>
    </svg>

    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg" class=hidden>
        <style>
            .al-code {
                font-family: monospace;
            }
        </style>
        <rect width=300 height=200 fill=white stroke="grey" stroke-width="5px" />

        <text class=al-code font-size=6 y=195 x=6 textWidth=65>With write barrier</text>

        <g transform="translate(25 10)">
            <text class="al-code" font-size=15 y="1em" dy="0em">foo = "first"</text>
            <text class="al-code" font-size=15 y="1em" dy="1em">foo = {}</text>
            <text class="al-code" font-size=15 y="1em" dy="2em">foo[0] = "second"</text>
            <text class="al-code" font-size=15 y="1em" dy="3em"># GC step</text>
            <text class="al-code" font-size=15 y="1em" dy="4em">foo[1] = "third"</text>
            <text class="al-code" font-size=15 y="1em" dy="5em"># GC step</text>
        </g>

        <rect fill="#EE6677" width="6" height="6" rx="1" x="13" y="1.3em">
            <animate id=root12 attributeName=y dur="1.0s" values="2.6em;3.4em" fill=freeze calcMode=spline keySplines="0.23 0.73 0.7 1" />
        </rect>

        <text textAnchor=middle x=13 y=120 font-size=10 text-decoration=underline>Legend</text>
        <rect fill=white width="6" height="6" stroke=black rx="1" x="13" y="125" />
        <text class=al-code x=22 y=130 font-size=6>Not marked</text>
        <rect fill=#CCBB44 width="6" height="6" stroke=black rx="1" x="13" y=134 />
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propogate, then become marked.</text>
        <rect fill=#228833 width="6" height="6" stroke=black rx="1" x="13" y=143 />
        <text class=al-code x=22 y=148 font-size=6>Marked</text>

        <g>
            <rect width=50 height=20 x=190 y=20 fill=white stroke=black />
            <text class="al-code" x=215 y=32 font-size=11 textLength=40 text-anchor=middle>"first"</text>
        </g>

        <g>
            <rect width=50 height=20 x=230 y=60 fill=none stroke=black>
                <animate begin=root12.begin attributeName=fill values="#CCBB44; #CCBB44; #CCBB44; #228833" fill=freeze dur="3s" keyTimes="0;0.33;0.8;1" />
            </rect>
            <text class="al-code" x=255 y=73 font-size=11 text-anchor=middle>{}</text>
        </g>

        <g>
            <rect width=50 height=20 x=200 y=100 fill=white stroke=black>
                <animate attributeName=fill dur=1s values="white;#CCBB44" fill=freeze begin="root12.begin+1" />
                <animate attributeName=fill dur=1s values="#CCBB44;#228833" fill=freeze begin=root12.begin+3.1 />
            </rect>
            <text class="al-code" x=225 y=112 font-size=10 textLength=45 text-anchor=middle>"second"</text>
            <line x1=225 y1=100 x2=255 y2=80 stroke=black>
                <animate begin=root12.begin attributeName=stroke values="black; #CCBB44; #CCBB44; black" fill=freeze dur="3s" keyTimes="0;0.33;0.66;1" />
                <animate begin=root12.begin attributeName=stroke-width values="1; 3; 3; 1" fill=freeze dur="3s" keyTimes="0;0.33;0.66;1" />
            </line>
        </g>


        <g opacity=0>
            <rect width=50 height=20 x=238 y=130 fill=white stroke=black />
            <text class="al-code" x=263 y=143 font-size=10 textLength=40 text-anchor=middle>"third"</text>
            <line x1=263 y1=130 x2=255 y2=80 stroke=black />
        </g>


        <text class="al-code" font-size="10" text-anchor="middle" x="150" y="153" dy=1em>GC runs and finishes marking {}</text>

        <svg viewBox="0 0 100 20" width=100 height=20 y=170 x=101>
            <a xlink:href="#write-barrier-1">
                <rect y="2" x="5" width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x=13 font-size=10 dominant-baseline=middle text-anchor=middle textLength=10 fill=black>1</text>
            </a>
            <a xlink:href="#write-barrier-2" class="active-slide">
                <rect y=2 x=26 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="34" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>2</text>
            </a>
            <a xlink:href="#write-barrier-3">
                <rect y=2 x=47 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="55" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>3</text>
            </a>
            <a xlink:href="#write-barrier-4">
                <rect y=2 x=68 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="76" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>4</text>
            </a>
        </svg>
    </svg>

    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg" class=hidden>
        <style>
            .al-code {
                font-family: monospace;
            }
        </style>
        <rect width=300 height=200 fill=white stroke="grey" stroke-width="5px" />

        <text class=al-code font-size=6 y=195 x=6 textWidth=65>With write barrier</text>

        <g transform="translate(25 10)">
            <text class="al-code" font-size=15 y="1em" dy="0em">foo = "first"</text>
            <text class="al-code" font-size=15 y="1em" dy="1em">foo = {}</text>
            <text class="al-code" font-size=15 y="1em" dy="2em">foo[0] = "second"</text>
            <text class="al-code" font-size=15 y="1em" dy="3em"># GC step</text>
            <text class="al-code" font-size=15 y="1em" dy="4em">foo[1] = "third"</text>
            <text class="al-code" font-size=15 y="1em" dy="5em"># GC step</text>
        </g>

        <rect fill="#EE6677" width="6" height="6" rx="1" x="13" y="1.3em">
            <animate id=root13 attributeName=y dur="1.0s" values="3.4em;4.3em" fill=freeze calcMode=spline keySplines="0.23 0.73 0.7 1" />
        </rect>

        <text textAnchor=middle x=13 y=120 font-size=10 text-decoration=underline>Legend</text>
        <rect fill=white width="6" height="6" stroke=black rx="1" x="13" y="125" />
        <text class=al-code x=22 y=130 font-size=6>Not marked</text>
        <rect fill=#CCBB44 width="6" height="6" stroke=black rx="1" x="13" y=134 />
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propogate, then become marked.</text>
        <rect fill=#228833 width="6" height="6" stroke=black rx="1" x="13" y=143 />
        <text class=al-code x=22 y=148 font-size=6>Marked</text>

        <g>
            <rect width=50 height=20 x=190 y=20 fill=white stroke=black />
            <text class="al-code" x=215 y=32 font-size=11 textLength=40 text-anchor=middle>"first"</text>
        </g>

        <g>
            <rect width=50 height=20 x=230 y=60 fill=#228833 stroke=black>
            </rect>
            <text class="al-code" x=255 y=73 font-size=11 text-anchor=middle>{}</text>
        </g>

        <g>
            <rect width=50 height=20 x=200 y=100 fill=#228833 stroke=black>
            </rect>
            <text class="al-code" x=225 y=112 font-size=10 textLength=45 text-anchor=middle>"second"</text>
            <line x1=225 y1=100 x2=255 y2=80 stroke=black>
            </line>
        </g>


        <g opacity=0>
            <rect width=50 height=20 x=238 y=130 fill=white stroke=black>
                <animate attributeName=fill dur=0.8s values="white;#CCBB44" begin=root13.begin+1.6 fill=freeze />
            </rect>
            <text class="al-code" x=263 y=143 font-size=10 textLength=40 text-anchor=middle>"third"</text>
            <animate attributeName=opacity dur=1.0s values="0;1" fill=freeze begin=root13.begin />
            <line x1=263 y1=130 x2=255 y2=80 stroke=black>
                <animate attributeName=stroke-width dur=0.8s values="1;2.5;1" begin=root13.begin+1 fill=freeze />
            </line>
        </g>


        <text class="al-code" font-size="10" text-anchor="middle" x="150" y="153" dy=1em>Code adds new reference with write barrier</text>

        <svg viewBox="0 0 100 20" width=100 height=20 y=170 x=101>
            <a xlink:href="#write-barrier-1">
                <rect y="2" x="5" width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x=13 font-size=10 dominant-baseline=middle text-anchor=middle textLength=10 fill=black>1</text>
            </a>
            <a xlink:href="#write-barrier-2">
                <rect y=2 x=26 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="34" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>2</text>
            </a>
            <a xlink:href="#write-barrier-3" class="active-slide">
                <rect y=2 x=47 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="55" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>3</text>
            </a>
            <a xlink:href="#write-barrier-4">
                <rect y=2 x=68 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="76" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>4</text>
            </a>
        </svg>
    </svg>

    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg" class=hidden>
        <style>
            .al-code {
                font-family: monospace;
            }
        </style>
        <rect width=300 height=200 fill=white stroke="grey" stroke-width="5px" />

        <text class=al-code font-size=6 y=195 x=6 textLength=65>With write barrier</text>

        <g transform="translate(25 10)">
            <text class="al-code" font-size=15 y="1em" dy="0em">foo = "first"</text>
            <text class="al-code" font-size=15 y="1em" dy="1em">foo = {}</text>
            <text class="al-code" font-size=15 y="1em" dy="2em">foo[0] = "second"</text>
            <text class="al-code" font-size=15 y="1em" dy="3em"># GC step</text>
            <text class="al-code" font-size=15 y="1em" dy="4em">foo[1] = "third"</text>
            <text class="al-code" font-size=15 y="1em" dy="5em"># GC step</text>
        </g>

        <rect fill="#EE6677" width="6" height="6" rx="1" x="13" y="1.3em">
            <animate id=root14 attributeName=y dur="1.0s" values="4.3em;5.0em" fill=freeze calcMode=spline keySplines="0.23 0.73 0.7 1" />
        </rect>

        <text textAnchor=middle x=13 y=120 font-size=10 text-decoration=underline>Legend</text>
        <rect fill=white width="6" height="6" stroke=black rx="1" x="13" y="125" />
        <text class=al-code x=22 y=130 font-size=6>Not marked</text>
        <rect fill=#CCBB44 width="6" height="6" stroke=black rx="1" x="13" y=134 />
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propogate, then become marked.</text>
        <rect fill=#228833 width="6" height="6" stroke=black rx="1" x="13" y=143 />
        <text class=al-code x=22 y=148 font-size=6>Marked</text>

        <g>
            <rect width=50 height=20 x=190 y=20 fill=white stroke=black />
            <text class="al-code" x=215 y=32 font-size=11 textLength=40 text-anchor=middle>"first"</text>
        </g>

        <g>
            <rect width=50 height=20 x=230 y=60 fill=#228833 stroke=black>
            </rect>
            <text class="al-code" x=255 y=73 font-size=11 text-anchor=middle>{}</text>
        </g>

        <g>
            <rect width=50 height=20 x=200 y=100 fill=#228833 stroke=black>
            </rect>
            <text class="al-code" x=225 y=112 font-size=10 textLength=45 text-anchor=middle>"second"</text>
            <line x1=225 y1=100 x2=255 y2=80 stroke=black>
            </line>
        </g>

        <g>
            <rect width=50 height=20 x=238 y=130 fill=#CCBB44 stroke=black>
                <animate attributeName=fill dur=1s values="#CCBB44;#228833" begin=root14.begin+0.5 fill=freeze />
            </rect>
            <text class="al-code" x=263 y=143 font-size=10 textLength=40 text-anchor=middle>"third"</text>
            <line x1=263 y1=130 x2=255 y2=80 stroke=black />
        </g>


        <text class="al-code" font-size="10" text-anchor="middle" x="150" y="153" dy=1em>GC completes correctly</text>

        <svg viewBox="0 0 100 20" width=100 height=20 y=170 x=101>
            <a xlink:href="#write-barrier-1">
                <rect y="2" x="5" width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x=13 font-size=10 dominant-baseline=middle text-anchor=middle textLength=10 fill=black>1</text>
            </a>
            <a xlink:href="#write-barrier-2">
                <rect y=2 x=26 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="34" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>2</text>
            </a>
            <a xlink:href="#write-barrier-3">
                <rect y=2 x=47 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="55" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>3</text>
            </a>
            <a xlink:href="#write-barrier-4" class="active-slide">
                <rect y=2 x=68 width="16" height="16" stroke=black fill=white></rect>
                <text class="al-code" y="10" x="76" font-size=10 dominant-baseline=middle text-anchor=middle textLength=10>4</text>
            </a>
        </svg>
    </svg>
</div>

<script>
(function(){
    function setupButtons(root) {
        const slideCount = root.children.length;
        let buttons = root.querySelectorAll("svg>a");
        for (let button of buttons) {
            // These are SVGAElement.
            let href = button.href.baseVal;
            let lastChar = href.charAt(href.length-1);
            let lastCharAsNum = Number.parseInt(lastChar);
            let transitionTo = lastCharAsNum-1;
            if (transitionTo >= 0 && transitionTo < slideCount) {
                button.addEventListener("click", function(clickEv) {
                    clickEv.preventDefault(); // To prevent changing the URI fragment.
                    for (let idx=0; idx<slideCount; idx++) {
                        let slideElement = root.children[idx];
                        if (idx == transitionTo) {
                            slideElement.classList.remove("hidden");
                        } else if (!slideElement.classList.contains("hidden")) {
                            slideElement.classList.add("hidden");
                        }
                    }
                    root.querySelectorAll("animate").forEach(e => {
                        // Hack for restarting the SVG animation timeline.
                        // Reinsert all elements then kick the animation on the syncbase element
                        // that all the animate elements are based off of.
                        let parent = e.parentNode;
                        parent.removeChild(e);
                        parent.insertAdjacentHTML("beforeend", e.outerHTML);
                        if (e.id.startsWith("root")) {
                            parent.children[parent.children.length-1].beginElement();
                        }
                    });
                });
            } else {
                console.error("failed finding index for transition button", button);
            }
        }
    }
    setupButtons(document.getElementById("al-svg-1"));
    setupButtons(document.getElementById("al-svg-2"));
    setupButtons(document.getElementById("al-svg-3"));
    setupButtons(document.getElementById("al-svg-4"));
})();
</script>
{{</html>}}

