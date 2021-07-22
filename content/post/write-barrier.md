---
title: "Write Barrier"
date: 2021-07-22T06:27:34-04:00
draft: true
---

{{<html>}}
<div id="al-svg-1">
    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg">
        <style>
            .al-code {
                font-family: monospace;
            }
        </style>
        <rect width=300 height=200 fill=none stroke="grey" stroke-width="5px" />

        <g transform="translate(25 10)">
            <text class="al-code" font-size=15 y="1em" dy="0em">foo = "first"</text>
            <text class="al-code" font-size=15 y="1em" dy="1em">puts("foo")</text>
            <text class="al-code" font-size=15 y="1em" dy="2em">foo = {}</text>
            <text class="al-code" font-size=15 y="1em" dy="3em" >foo[0] = "second"</text>
            <text class="al-code" font-size=15 y="1em" dy="4em" >foo[1] = "third"</text>
            <text class="al-code" font-size=15 y="1em" dy="5em">GC.start</text>
        </g>

        <rect fill="#EE6677" width="6" height="6" rx="1" x="13" y="1.3em">
            <animate attributeName=y dur="1.0s" values="1.5em;4.3em" fill=freeze calcMode=spline keySplines="0.23 0.73 0.7 1" />
        </rect>

        <g>
            <rect width=50 height=20 x=180 y=20 fill=white stroke=black />
            <text class="al-code" x=205 y=32 font-size=11 textLength=40 text-anchor=middle>"first"</text>
            <animate attributeName=opacity dur=1.7s values="0;1" />
        </g>

        <g>
            <rect width=50 height=20 x=230 y=60 fill=white stroke=black />
            <text class="al-code" x=255 y=73 font-size=11 text-anchor=middle>{}</text>
            <animate attributeName=opacity dur=1.7s values="0;1" begin=0.1 />
        </g>


        <g>
            <rect width=50 height=20 x=200 y=100 fill=white stroke=black />
            <text class="al-code" x=225 y=112 font-size=10 textLength=45 text-anchor=middle>"second"</text>
            <line x1=225 y1=100 x2=255 y2=80 stroke=black />
            <animate attributeName=opacity dur=1.7s values="0;1" begin=0.2 />
        </g>

        <g>
        <rect width=50 height=20 x=238 y=130 fill=white stroke=black />
        <text class="al-code" x=263 y=143 font-size=10 textLength=40 text-anchor=middle>"third"</text>
        <line x1=263 y1=130 x2=255 y2=80 stroke=black />
            <animate attributeName=opacity dur=1.7s values="0;1" begin=0.3 />
        </g>


        <text class="al-code" font-size="10" text-anchor="middle" x="150" y="153">Code allocate objects</text>

        <a xlink:href="#stop-the-world-1">
            <rect y="170" x="121" width="16" height="16" fill=white stroke=#4477AA></rect>
            <text class="al-code" y="182" x="129" font-size=10 text-anchor=middle textLength=10 fill=#4477AA>1</text>
        </a>
        <rect y="170" x="142" width="16" height="16" stroke=black fill=none></rect>
        <text class="al-code" y="182" x="150" font-size=10 text-anchor=middle textLength=10>2</text>
        <rect y="170" x="163" width="16" height="16" stroke=black fill=none></rect>
        <text class="al-code" y="182" x="171" font-size=10 text-anchor=middle textLength=10>3</text>

    </svg>

    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg">
        <style>
            .al-code {
                font-family: monospace;
            }
        </style>
        <rect width=300 height=200 fill=none stroke="grey" stroke-width="5px" />

        <g transform="translate(25 10)">
            <text class="al-code" font-size=15 y="1em" dy="0em">foo = "first"</text>
            <text class="al-code" font-size=15 y="1em" dy="1em">puts("foo")</text>
            <text class="al-code" font-size=15 y="1em" dy="2em">foo = {}</text>
            <text class="al-code" font-size=15 y="1em" dy="3em" >foo[0] = "second"</text>
            <text class="al-code" font-size=15 y="1em" dy="4em" >foo[1] = "third"</text>
            <text class="al-code" font-size=15 y="1em" dy="5em">GC.start</text>
        </g>

        <rect fill="#EE6677" width="6" height="6" rx="1" x="13" y="1.3em">
            <animate attributeName=y dur="1.0s" values="4.3em;5.1em" fill=freeze calcMode=spline keySplines="0.23 0.73 0.7 1" />
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
                <animate attributeName=fill values="#CCBB44; #CCBB44; #CCBB44; #228833" fill=freeze dur="3s" keyTimes="0;0.33;0.8;1" />
            </rect>
            <text class="al-code" x=255 y=73 font-size=11 text-anchor=middle>{}</text>
        </g>


        <rect width=50 height=20 x=200 y=100 fill=white stroke=black>
            <animate attributeName=fill dur=1s values="white;#CCBB44" fill=freeze begin=1 />
            <animate attributeName=fill dur=1s values="#CCBB44;#228833" fill=freeze begin=3.5 />
        </rect>
        <text class="al-code" x=225 y=112 font-size=10 textLength=45 text-anchor=middle>"second"</text>
        <line x1=225 y1=100 x2=255 y2=80 stroke=black>
            <animate attributeName=stroke values="black; #CCBB44; #CCBB44; black" fill=freeze dur="3s" keyTimes="0;0.33;0.66;1" />
            <animate attributeName=stroke-width values="1; 3; 3; 1" fill=freeze dur="3s" keyTimes="0;0.33;0.66;1" />
        </line>

        <rect width=50 height=20 x=238 y=130 fill=white stroke=black>
            <animate attributeName=fill dur=1s values="white;#CCBB44" fill=freeze begin=1 />
            <animate attributeName=fill dur=1s values="#CCBB44;#228833" fill=freeze begin=3.5 />
        </rect>
        <text class="al-code" x=263 y=143 font-size=10 textLength=40 text-anchor=middle>"third"</text>
        <line x1=263 y1=130 x2=255 y2=80 stroke=black>
            <animate attributeName=stroke values="black; #CCBB44; #CCBB44; black" fill=freeze dur="3s" keyTimes="0;0.33;0.66;1" />
            <animate attributeName=stroke-width values="1; 3; 3; 1" fill=freeze dur="3s" keyTimes="0;0.33;0.66;1" />
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

    <svg viewBox="0 0 300 200" xmlns="http://www.w3.org/2000/svg">
        <style>
            .al-code {
                font-family: monospace;
            }
        </style>
        <rect width=300 height=200 fill=none stroke="grey" stroke-width="5px" />

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
            <rect y="170" x="142" width="16" height="16" stroke=#4477AA fill=white></rect>
            <text class="al-code" y="182" x="150" font-size=10 text-anchor=middle textLength=10 fill=#4477AA>2</text>
        </a>
        <a xlink:href="#stop-the-world-3">
            <rect y="170" x="163" width="16" height="16" stroke=black fill=none></rect>
            <text class="al-code" y="182" x="171" font-size=10 text-anchor=middle textLength=10>3</text>
        </a>

    </svg>
</div>
{{</html>}}

