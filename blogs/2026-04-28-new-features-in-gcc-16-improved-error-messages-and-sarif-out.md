---
title: "New features in GCC 16: Improved error messages and SARIF output"
url: "https://developers.redhat.com/articles/2026/04/28/gcc-16-improved-error-messages-sarif-output"
date: "Tue, 28 Apr 2026 07:16:25 +0000"
author: "David Malcolm"
feed_url: "https://developers.redhat.com/blog/feed"
---
<p>I work at Red Hat on the <a href="https://gcc.gnu.org/">GNU Compiler Collection (GCC)</a>. GCC 16 is about to be released, so I'm sharing some of the new features I worked on this year. Some changes are visible to users, while others improve the system more subtly.</p><h2>New C++ error improvements</h2><p>A well-known challenge for C++ developers is the readability of template-related error messages. C++ compilers tend to either provide too little information or spew screenfuls of text at you. Either way, the errors can be difficult to decipher.</p><p>GCC error messages have a hierarchical structure to them. In GCC 15, I <a href="https://developers.redhat.com/articles/2025/04/10/6-usability-improvements-gcc-15?source=sso#2__a_new_look_for_c___template_errors">added an experimental option</a> that shows this structure as a collection of nested bullet points.</p><p>In GCC 16, this behavior is now the default. You can return to the previous behavior using <a href="https://gcc.gnu.org/onlinedocs/gcc/Diagnostic-Message-Formatting-Options.html#index-fno-diagnostics-show-nesting">-fno-diagnostics-show-nesting</a> or <a href="https://gcc.gnu.org/onlinedocs/gcc/Diagnostic-Message-Formatting-Options.html#index-fdiagnostics-plain-output">-fdiagnostics-plain-output</a>. I fixed several bugs and made use of the hierarchical structure in more places. For example, it is easy to get declarations and definitions out of sync when manually adding <code>const</code> to a parameter:</p><pre><code class="language-cpp">class foo
{
  public:
    void test(int i, int j, void *ptr, int k);
};
    
// Wrong "const"-ness of param 3.
void foo::test(int i, int j, const void *ptr, int k)
{
}</code></pre><p>In <a href="https://godbolt.org/z/9qMxEz3Pe">GCC 15</a>, we emitted the following output:</p><pre><code class="language-output">&lt;source&gt;:8:6: error: no declaration matches 'void foo::test(int, int, const void*, int)'
    8 | void foo::test(int i, int j, const void *ptr, int k)
      |      ^~~
&lt;source&gt;:4:10: note: candidate is: 'void foo::test(int, int, void*, int)'
    4 |     void test(int i, int j, void *ptr, int k);
      |          ^~~~
&lt;source&gt;:1:7: note: 'class foo' defined here
    1 | class foo
      |       ^~~</code></pre><p>In <a href="https://godbolt.org/z/ceMTc8zTj">GCC 16</a>, we now emit this:</p><pre><code class="language-output">&lt;source&gt;:8:6: error: no declaration matches 'void foo::test(int, int, const void*, int)'
    8 | void foo::test(int i, int j, const void *ptr, int k)
      |      ^~~
  • there is 1 candidate
    • candidate is: 'void foo::test(int, int, void*, int)'
      &lt;source&gt;:4:10:
          4 |     void test(int i, int j, void *ptr, int k);
            |          ^~~~
      • parameter 3 of candidate has type 'void*'...
        &lt;source&gt;:4:35:
            4 |     void test(int i, int j, void *ptr, int k);
              |                             ~~~~~~^~~
      • ...which does not match type 'const void*'
        &lt;source&gt;:8:42:
            8 | void foo::test(int i, int j, const void *ptr, int k)
              |                              ~~~~~~~~~~~~^~~
&lt;source&gt;:1:7: note: 'class foo' defined here
    1 | class foo
      |       ^~~</code></pre><p>This pinpoints the exact location of the problem. Use <a href="https://godbolt.org/z/v1hrbWf77">this Compiler Explorer link</a> to see how color highlights and contrasts mismatched types in both the messages and the quoted source code.</p><h2>Updated SARIF machine-readable output</h2><p>By default, GCC writes its diagnostics (errors and warnings) as text to <code>stderr</code>. Parsing this output with regular expressions has become difficult as the compiler's capabilities have grown. In GCC 13, I added the ability to write diagnostics in machine-readable form using the <a href="https://gcc.gnu.org/wiki/SARIF">Static Analysis Results Interchange Format (SARIF)</a>. This JSON-based format allows us to separate the data of the diagnostic from the way the diagnostic is presented.</p><p>GCC 16 includes several improvements to the generated SARIF output. For example, when reporting a missing <code>return *this</code> in an assignment operator:</p><pre><code class="language-cpp">namespace foo { 
namespace bar { 
class foo { 
  foo&amp;
  operator= (const foo &amp;other)
  {
    m_val = other.m_val;
  }
  int m_val;
};
} // namespace bar
} // namespace foo  </code></pre><p>The <a href="https://godbolt.org/z/4jTv8ocv1">SARIF output</a> now captures the nested structure of logical locations. This allows a SARIF viewer to filter for diagnostics within the <code>foo::bar</code> namespace:</p><pre><code class="language-javascript">           "logicalLocations": [{"name": "foo",
                                 "fullyQualifiedName": "foo",
                                 "kind": "namespace",
                                 "index": 0},
                                {"name": "bar",
                                 "fullyQualifiedName": "foo::bar",
                                 "kind": "namespace",
                                 "parentIndex": 0,
                                 "index": 1},
                                {"name": "baz",
                                 "kind": "type",
                                 "parentIndex": 1,
                                 "index": 2},
                                {"name": "operator=",
                                 "fullyQualifiedName": "foo::bar::baz::operator=",
                                 "decoratedName": "_ZN3foo3bar3bazaSERKS1_",
                                 "kind": "function",
                                 "parentIndex": 2,
                                 "index": 3}]</code></pre><p>GCC 16 also adds data to SARIF output to better express non-standard control flow (such as exception-handling and <code>longjmp</code>) within code paths. This data is included in the upcoming <a href="https://github.com/oasis-tcs/sarif-spec/wiki/What%27s-new-in-SARIF-2.2">SARIF 2.2 standard</a>.</p><h2>New HTML output option</h2><p>In GCC 15, I added <a href="https://gcc.gnu.org/onlinedocs/gcc/Diagnostic-Message-Formatting-Options.html#index-fdiagnostics-add-output">-fdiagnostics-add-output=</a> to allow for multiple kinds of diagnostic output simultaneously. Plain text output has limitations, so GCC 16 includes a new <a href="https://gcc.gnu.org/onlinedocs/gcc/Diagnostic-Message-Formatting-Options.html#index-fdiagnostics-add-output_003dexperimental-html">experimental-html</a> option.</p><p>Figure 1 shows the first example using <code>-fdiagnostics-add-output=experimental-html</code>.</p><figure class="rhd-u-has-filter-caption">
<article class="media media--type-image media--view-mode-embedded">
  
      
        <div class="field__item">  <img alt="Screeenshot of Firefox showing HTML output" height="767" src="https://developers.redhat.com/sites/default/files/screenshot_from_2026-04-24_11-52-50.png" width="739" />

</div>
  </article>

<figcaption class="rhd-c-caption">Figure 1: An experimental HTML diagnostic in GCC 16 showing a "no declaration matches" error with highlighted code snippets and callouts.</figcaption>
</figure>
<p>You can see the full generated page <a href="https://dmalcolm.fedorapeople.org/gcc/2026-04-24/bad-def.cc.html">here</a>.</p><p>As the name suggests, this feature is experimental, but I've already found it helpful for debugging the <a href="https://gcc.gnu.org/wiki/StaticAnalyzer">GCC built-in static analyzer</a>. When you enable the tool with the <a href="https://developers.redhat.com/blog/2020/03/26/static-analysis-in-gcc-10">-fanalyzer option</a>, it explores interprocedural paths through your source code to find bugs at compile time. I often need to debug fiddly issues in this code, and the more visualization the better. The HTML output displays nested stack frames in an execution path, using drop shadows to represent the stack visually (Figure 2).</p><figure class="rhd-u-has-filter-caption">
<article class="media media--type-image media--view-mode-embedded">
  
      
        <div class="field__item">  <img alt="Screenshot of Firefox showing visualization of interprocedural control flow" height="1182" src="https://developers.redhat.com/sites/default/files/screenshot_from_2026-04-24_12-20-03.png" width="1079" />

</div>
  </article>

<figcaption class="rhd-c-caption">Figure 2: Experimental HTML output from the GCC static analyzer, illustrating a 26-event execution path with nested frames and visual drop shadows to represent the call stack.</figcaption>
</figure>
<p>The full example is <a href="https://dmalcolm.fedorapeople.org/gcc/2026-04-24/linked-list.c.html">here</a>. This version also includes an easter egg (generated via <code>-fdiagnostics-add-output=experimental-html:show-state-diagrams=yes</code>). If you press <strong>j</strong> and <strong>k</strong>, you can move forward and backward through the path, with diagrams showing the predicted state of memory at each event, and what pointers are pointing to what buffers (Figure 3).</p><figure class="rhd-u-has-filter-caption">
<article class="media media--type-image media--view-mode-embedded">
  
      
        <div class="field__item">  <img alt="Screenshot of Firefox showing state visualization" height="958" src="https://developers.redhat.com/sites/default/files/screenshot_from_2026-04-24_12-13-07.png" width="1170" />

</div>
  </article>

<figcaption class="rhd-c-caption">Figure 3: An experimental GCC state diagram visualizing the heap and stack memory transitions, illustrating a pointer referencing a freed buffer.</figcaption>
</figure>
<h2>Static analyzer improvements</h2><p>The visualization made it easier to spot and fix problems in the analyzer, leading to several internal improvements in GCC 16. The analyzer's core data structure for tracking code (the "supergraph") had become difficult to work with, with various dark corners for bugs to hide in. I've <a href="https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=0b786d961d4426db15b3f681f4e7c6fcaeb296c2">rewritten it in GCC 16</a>. The new code has much clearer separation of concerns (between <em>places in the user's code</em> versus <em>operations</em> that occur on the transitions between these places), and I'm already finding it makes it easier to add new features.</p><p>I also updated the data structure that tracks simulated memory buffer contents, replacing a rather clunky hashing approach with a simple <code>std::map</code> from bit ranges to contents. The <a href="https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=310a70ef6db45d2fd574daa77b5128cd1f4edbce">new approach</a> is both easier to understand and faster.</p><p>My colleague Andrew MacLeod has spent several years on a <a href="https://developers.redhat.com/blog/2021/04/28/value-range-propagation-in-gcc-with-project-ranger">project called Ranger</a> to improve how GCC tracks properties of values in the user's code for use by the optimizer. This covers things such as knowing whether a given integer is in the valid range to be used as an array index, or whether individual bits are known to be true or false. In GCC 16 I've started wiring up <code>-fanalyzer</code> to these data structures. Like the above changes, this is unlikely to be directly visible, but should lead to more accurate analysis and fewer false positives.</p><p>Before GCC 16, <code>-fanalyzer</code> only worked on C code; running it on C++ code often produced irrelevant results. The problem was that my code was ignoring how GCC internally represents (a) exception-handling, leading to it inventing impossible paths through the code, and (b) C++'s Named Return Value Optimization (NRVO), leading to lots of false reports about supposed memory leaks.</p><p>I have good news and bad news here. The good news: In GCC 16, I implemented exception handling and the NRVO, allowing <code>-fanalyzer</code> to work with C++ code.&nbsp;</p><p>The bad news is that the feature is currently limited to small examples. Running it on complex code might cause scaling issues where the analyzer spends its entire analysis budget on a small fraction of the code and gives up, burning CPU cycles without generating useful results. False positives have been replaced by false negatives, and so I still can't recommend using it on C++ code. It's better to be correct than to be fast, and I'm looking at ways of scaling things up for GCC 17 in the hope of making <code>-fanalyzer</code> be practical for use on production C++.</p><h2>Try GCC 16</h2><p>These features are just a small sample of the <a href="https://gcc.gnu.org/gcc-16/changes.html">many improvements in GCC 16</a>, which is about to be officially released upstream. You can <a href="https://godbolt.org/z/sPf7erEbq">try the new version now in Compiler Explorer</a>; as I write this, it's listed as <strong>GCC trunk</strong>. Putting my downstream hat on, you can also try it in Fedora 44. Have fun!</p>

The post <a href="https://developers.redhat.com/articles/2026/04/28/gcc-16-improved-error-messages-sarif-output" title="New features in GCC 16: Improved error messages and SARIF output">New features in GCC 16: Improved error messages and SARIF output</a> appeared first on <a href="https://developers.redhat.com/blog" title="Red Hat Developer">Red Hat Developer</a>.
<br /><br />
