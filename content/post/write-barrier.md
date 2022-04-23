---
title: "Write barriers and an old Ruby bug"
date: 2021-07-22T06:27:34-04:00
summary: >-
  Learn about the role write barriers play in garbage collection algorithms through
  CRuby's implementation and example animations.
---

Here is a program that crashes Ruby 2.7.0:
```ruby
hash = { a: Object.new }
GC.start
hash.transform_values!{ Object.new }
GC.start(full_mark: false)
GC.start
```

The crash message is scary and subtly hints at the presence of a bug:
```
[BUG] try to mark T_NONE object
```
In this post, I will explain the reason behind the crash and suggest some ways to avoid similar issues in the future.

## Is it a bug in the garbage collector?

Since the program crashes without the help of third party C extensions and crashes even when Ruby is launched with `--disable-gems`, it's clear that the problem is in the Ruby runtime. Looking at the C stack trace, we can see that `gc.c` initiates the crash. How can we prove beyond a reasonable doubt that the GC itself is guilty of this crash? It turns out, there is a method, `GC.verify_internal_consistency` that can help us out.

Adding a call to this method right after the call to `transform_values!` in our reproducer, we get a different message:
```
verify_internal_consistency_reachable_i: WB miss (O->Y) T_HASH -> T_OBJECT
crash.rb:4: [BUG] gc_verify_internal_consistency: found internal inconsistency.
```
Okay, it looks like some invariant that the GC cares about is not upheld, but what is a "WB miss"?

## Incremental mark and sweep

WB stands for write barrier so `gc_verify_internal_consistency` is telling us that there is a "write barrier miss". To understand what write barriers are and what they are for, we first need to understand how the GC functions. Once upon a time, Ruby had a stop-the-world mark-and-sweep GC. "Stop-the-world" sounds grandiose, but it basically means no Ruby code is running while the GC is doing work. I made some animated slides to explain the stop-the-world GC algorithm. [^1]

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
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propagate, then become marked.</text>
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
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propagate, then become marked.</text>
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
{{</html>}}

This works okay, with one disadvantage: when there are many objects alive, the GC could pause Ruby code execution for an unacceptably long amount of time. Pausing for just 17 milliseconds in a video game could be devastating as that can easily make the game miss its frame rate target.

To deal with this problem, we could break up the work the GC has to do into chunks and interleave them with Ruby code execution. To go back to the video game example, instead of pausing for say 17 milliseconds a single time, we pause for 1 millisecond 17 times, allowing Ruby code to execute between pauses. If the game has 17 milliseconds to render each frame, instead of taking up 17 milliseconds once in a while, blowing through the frame time budget, the GC now takes up 1 millisecond once in a while. The GC potentially takes longer to release garbage objects, because the total amount of work is still the same as before, but that should be okay for a lot of applications.

{{<html>}}
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
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propagate, then become marked.</text>
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
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propagate, then become marked.</text>
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
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propagate, then become marked.</text>
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
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propagate, then become marked.</text>
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
{{</html>}}

## Write barriers

Great, we now have an incremental GC but we have also introduced a new problem. What happens if we finish marking an object `O`, let Ruby code execute, and the Ruby code associates `O` with an object `J` that would otherwise be unreachable?

{{<html>}}
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
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propagate, then become marked.</text>
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
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propagate, then become marked.</text>
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
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propagate, then become marked.</text>
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
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propagate, then become marked.</text>
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
{{</html>}}

We know `J` is alive since `O` is alive and all objects referenced by alive objects are also alive. However `O` is already marked, so the GC will not mark it again to see that `O` now refers to an object it didn't see the first time `O` was marked. Nevertheless, the GC cannot treat `J` as garbage. We need some way to address this problem.

We could make the Ruby code inform the GC every time it associates an object with another one. This way, when the Ruby code makes changes to an object the GC already marked the GC could be aware of objects that it potentially was unaware of at the time of marking:

{{<html>}}
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
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propagate, then become marked.</text>
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
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propagate, then become marked.</text>
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
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propagate, then become marked.</text>
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
        <text class=al-code x=22 y=139 font-size=6>Yet to be marked. Propagate, then become marked.</text>
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

The write barrier is a piece of code that informs the GC every time an object starts to reference a new object. In addition to facilitating incremental GC, write barriers also help with generational GC.

## How to "miss" a write barrier

Going back to our crash reproducer, we can now guess that `Hash#transform_values!` is missing a write barrier somewhere. After all, it's the only thing that can mutate the object reference graph after the GC runs. [This indeed was the problem](https://github.com/ruby/ruby/pull/2964).

There are many garbage collected languages that ship with compilers that can automatically insert write barriers. CRuby does not have that luxury, however, and it is up to the developers to manually insert write barriers where necessary. In this particular case a write barrier was present but erroneously removed.


## Consequences

Missing write barriers can make the GC collect live objects and lead to [use-after-free](https://en.wikipedia.org/w/index.php?title=Use_after_free). This can lead to crashes like we have seen, or worse, silent data corruption. The following program demonstrates data corruption:

```ruby
hash = { a: Object.new }
GC.start
hash.transform_values!{ 2 ** 99 }
p hash
GC.start(full_mark: false)
p hash

=begin
$ ruby -v corrupt.rb
ruby 2.7.0p0 (2019-12-25 revision 647ee6f091) [x86_64-darwin19]
{:a=>633825300114114700748351602688}
{:a=>":a"}
=end
```
Running the GC should not have visible impact on live objects.

## Possible ways to prevent this in the future

To quote Koichi Sasada in a [paper](https://www.atdot.net/~ko1/activities/rgengc_ismm.pdf) about CRuby's write barriers:

> In general, inserting WBs correctly is a difficult and time-consuming task because WB-related bugs cause critical issues and it is difficult to debug them.

Indeed, write barrier bugs can be very elusive. I certainly don't put it past myself to accidentally introduce one. They are hard to spot by testing. To illustrate, try removing or changing lines that start with `GC` in the reproducer -- a lot of the time the program doesn't crash and appears to work correctly.

I'm going to suggest some ways that can make forgetting to insert and/or accidentally removing write barriers more difficult.

### Education and code reviews

It's easy to forget to check for write barriers because a lot of the time one can compose the desired code change from pre-existing functions that already deal with write barriers correctly. Maybe the only thing that was stopping someone from catching this particular write barrier removal was a reminder to look for it. A gentle reminder on Github's pull request template could make sense.

I also think more people should understand write barriers enough that they know when to insert them. Write barriers form a very important contract between the GC and the rest of the virtual machine. People making changes should be aware of this contract.

### Static analysis

Humans are unreliable and can easily make mistakes. It would be ideal if we could run a program and get an assessment as to whether a write barrier is missing. For this particular case, it seems reasonable to write a Clang-based static analyzer that could catch it. However, I don't know how useful that would be for unknown cases that might reveal themselves in the future. Also, the ergonomics of static analyzers are hard to get right. Among other potential issues, just a few false alarms can make them too annoying to use.

Nevertheless, maybe it's worth it to make a best-effort static analyzer to help new contributors and experienced ones alike.

## Summary

- Write barriers are an important part of Ruby's incremental GC that updates the GC's understanding of the object reference graph
- CRuby developers are responsible for manually inserting write barriers wherever necessary
- Failing to insert write barriers can cause catastrophic failures such as crashes and data corruption


## Footnotes

[^1]: This is a textbook tri-color mark algorithm. Typically, the three colors are white, grey, and black. I chose different colors in an effort to help [colorblind](https://davidmathlogic.com/colorblind) readers.
