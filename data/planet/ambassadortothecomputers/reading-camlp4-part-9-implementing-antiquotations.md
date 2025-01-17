---
title: 'Reading Camlp4, part 9: implementing antiquotations'
description: In this post I want to complicate the JSON quotation library from the
  previous post  by adding antiquotations.   AST with antiquotations   I...
url: http://ambassadortothecomputers.blogspot.com/2010/08/reading-camlp4-part-9-implementing.html
date: 2010-08-06T00:55:00-00:00
preview_image:
featured:
authors:
- ambassadortothecomputers
---

<p>In this post I want to complicate the JSON quotation library from the <a href="http://ambassadortothecomputers.blogspot.com/2010/08/reading-camlp4-part-8-implementing.html">previous post</a> by adding antiquotations.</p> 
<b>AST with antiquotations</b> 
<p>In order to support antiquotations we will need to make some changes to the AST. Here is the new AST type:</p> 
<div class="highlight"><pre><code class="ocaml">  <span class="k">type</span> <span class="n">t</span> <span class="o">=</span> 
      <span class="o">...</span> <span class="c">(* base types same as before *)</span> 
    <span class="o">|</span> <span class="nc">Jq_array</span>  <span class="k">of</span> <span class="n">t</span> 
    <span class="o">|</span> <span class="nc">Jq_object</span> <span class="k">of</span> <span class="n">t</span> 
  
    <span class="o">|</span> <span class="nc">Jq_colon</span>  <span class="k">of</span> <span class="n">t</span> <span class="o">*</span> <span class="n">t</span> 
    <span class="o">|</span> <span class="nc">Jq_comma</span>  <span class="k">of</span> <span class="n">t</span> <span class="o">*</span> <span class="n">t</span> 
    <span class="o">|</span> <span class="nc">Jq_nil</span> 
  
    <span class="o">|</span> <span class="nc">Jq_Ant</span>    <span class="k">of</span> <span class="nn">Loc</span><span class="p">.</span><span class="n">t</span> <span class="o">*</span> <span class="kt">string</span> 
</code></pre> 
</div> 
<p>Let&rsquo;s first consider <code>Jq_Ant</code>. Antiquotations <code>$tag:body$</code> are returned from the lexer as an <code>ANTIQUOT</code> token containing the (possibly empty) tag and the entire body (including nested quotations/antiquotations) as a string. In the parser, we deal only with the JSON AST, so we can&rsquo;t really do anything with an antiquotation but return it to the caller (wrapped in a <code>Jq_Ant</code>).</p> 
 
<p>The lifting functions generated by <code>Camlp4MetaGenerator</code> treat <code>Jq_Ant</code> (and any other constructor ending in <code>Ant</code>) specially: instead of</p> 
<div class="highlight"><pre><code class="ocaml">  <span class="o">|</span> <span class="nc">Jq_Ant</span> <span class="o">(</span><span class="n">loc</span><span class="o">,</span> <span class="n">s</span><span class="o">)</span> <span class="o">-&gt;</span> 
      <span class="o">&lt;:</span><span class="n">expr</span><span class="o">&lt;</span> <span class="nc">Jq_Ant</span> <span class="o">($</span><span class="n">meta_loc</span> <span class="n">loc</span><span class="o">$,</span> <span class="o">$</span><span class="n">meta_string</span> <span class="n">s</span><span class="o">$)</span> <span class="o">&gt;&gt;</span> 
</code></pre> 
</div> 
<p>they have</p> 
<div class="highlight"><pre><code class="ocaml">  <span class="o">|</span> <span class="nc">Jq_Ant</span> <span class="o">(</span><span class="n">loc</span><span class="o">,</span> <span class="n">s</span><span class="o">)</span> <span class="o">-&gt;</span> <span class="nc">ExAnt</span> <span class="o">(</span><span class="n">loc</span><span class="o">,</span> <span class="n">s</span><span class="o">)</span> 
</code></pre> 
</div> 
<p>Instead of lifting the constructor, they translate it directly to <code>ExAnt</code> (or <code>PaAnt</code>, depending on the context). We don&rsquo;t otherwise have locations in our AST, but <code>Jq_Ant</code> must take a <code>Loc.t</code> argument because <code>ExAnt</code> does. Later, when we walk the OCaml AST expanding antiquotations, it will be convenient to have them as <code>ExAnt</code> nodes rather than lifted <code>Jq_Ant</code> nodes.</p> 
 
<p>In addition to <code>Jq_Ant</code>, we have new <code>Jq_nil</code>, <code>Jq_comma</code>, and <code>Jq_colon</code> constructors, and we have replaced the lists in <code>Jq_array</code> and <code>Jq_object</code> with just <code>t</code>. The idea here is that in an antiquotation in an array, e.g.</p> 
<div class="highlight"><pre><code class="ocaml">  <span class="o">&lt;:</span><span class="n">json</span><span class="o">&lt;</span> <span class="o">[</span> <span class="mi">1</span><span class="o">,</span> <span class="bp">true</span><span class="o">,</span> <span class="o">$</span><span class="n">x</span><span class="o">$,</span> <span class="s2">&quot;foo&quot;</span> <span class="o">]</span> <span class="o">&gt;&gt;</span> 
</code></pre> 
</div> 
<p>we would like to be able to substitute any number of elements (including zero) into the array in place of <code>x</code>. If <code>Jq_array</code> took a list, we could substitute exactly one element only. So instead we build a tree out of <code>Jq_comma</code> and <code>Jq_nil</code> constructors; at any point in the tree we can substitute zero (<code>Jq_nil</code>), one (any other <code>t</code> constructor), or more than one (a <code>Jq_comma</code> subtree) elements. We recover a list by taking the fringe of the final tree. (In the <code>Jq_ast</code> module there are functions <code>t_of_list</code> and <code>list_of_t</code> which convert between these representations.) For objects, we use <code>Jq_colon</code> to associate a name with a value, then build a tree of name/value pairs the same way.</p> 
 
<p>While this AST meets the need, it is now possible to have ill-formed ASTs, e.g. a bare <code>Jq_nil</code>, or a <code>Jq_object</code> where the elements are not <code>Jq_colon</code> pairs, or where the first argument of <code>Jq_colon</code> is not a <code>Jq_string</code>. This is annoying, but it is hard to see how to avoid it without complicating the AST and making it more difficult to use antiquotations.</p> 
<b>Parsing antiquotations</b> 
<p>Here is the updated parser:</p> 
<div class="highlight"><pre><code class="ocaml">  <span class="nc">EXTEND</span> <span class="nc">Gram</span> 
    <span class="n">json</span><span class="o">:</span> <span class="o">[[</span> 
        <span class="o">...</span> <span class="c">(* base types same as before *)</span> 
  
      <span class="o">|</span> <span class="o">`</span><span class="nc">ANTIQUOT</span> 
          <span class="o">(</span><span class="s2">&quot;&quot;</span><span class="o">|</span><span class="s2">&quot;bool&quot;</span><span class="o">|</span><span class="s2">&quot;int&quot;</span><span class="o">|</span><span class="s2">&quot;flo&quot;</span><span class="o">|</span><span class="s2">&quot;str&quot;</span><span class="o">|</span><span class="s2">&quot;list&quot;</span><span class="o">|</span><span class="s2">&quot;alist&quot;</span> <span class="k">as</span> <span class="n">n</span><span class="o">,</span> <span class="n">s</span><span class="o">)</span> <span class="o">-&gt;</span> 
            <span class="nc">Jq_Ant</span> <span class="o">(_</span><span class="n">loc</span><span class="o">,</span> <span class="n">n</span> <span class="o">^</span> <span class="s2">&quot;:&quot;</span> <span class="o">^</span> <span class="n">s</span><span class="o">)</span> 
  
      <span class="o">|</span> <span class="s2">&quot;[&quot;</span><span class="o">;</span> <span class="n">es</span> <span class="o">=</span> <span class="nc">SELF</span><span class="o">;</span> <span class="s2">&quot;]&quot;</span> <span class="o">-&gt;</span> <span class="nc">Jq_array</span> <span class="n">es</span> 
      <span class="o">|</span> <span class="s2">&quot;{&quot;</span><span class="o">;</span> <span class="n">kvs</span> <span class="o">=</span> <span class="nc">SELF</span><span class="o">;</span> <span class="s2">&quot;}&quot;</span> <span class="o">-&gt;</span> <span class="nc">Jq_object</span> <span class="n">kvs</span> 
  
      <span class="o">|</span> <span class="n">e1</span> <span class="o">=</span> <span class="nc">SELF</span><span class="o">;</span> <span class="s2">&quot;,&quot;</span><span class="o">;</span> <span class="n">e2</span> <span class="o">=</span> <span class="nc">SELF</span> <span class="o">-&gt;</span> <span class="nc">Jq_comma</span> <span class="o">(</span><span class="n">e1</span><span class="o">,</span> <span class="n">e2</span><span class="o">)</span> 
      <span class="o">|</span> <span class="o">-&gt;</span> <span class="nc">Jq_nil</span> 
 
      <span class="o">|</span> <span class="n">e1</span> <span class="o">=</span> <span class="nc">SELF</span><span class="o">;</span> <span class="s2">&quot;:&quot;</span><span class="o">;</span> <span class="n">e2</span> <span class="o">=</span> <span class="nc">SELF</span> <span class="o">-&gt;</span> <span class="nc">Jq_colon</span> <span class="o">(</span><span class="n">e1</span><span class="o">,</span> <span class="n">e2</span><span class="o">)</span>  
    <span class="o">]];</span> 
  <span class="nc">END</span> 
</code></pre> 
</div> 
<p>We want to support several kinds of antiquotations: For individual elements, <code>$x$</code> (where <code>x</code> is a <code>t</code>), or <code>$bool:x$</code>, <code>$int:x$</code>, <code>$flo:x$</code>, or <code>$str:x$</code> (where <code>x</code> is an OCaml <code>bool</code>, <code>int</code>, <code>float</code>, or <code>string</code>); for these latter cases we need to wrap <code>x</code> in the appropriate <code>t</code> constructor. For lists of elements, <code>$list:x$</code> where <code>x</code> is a <code>t list</code>, and <code>$alist:x$</code> where <code>x</code> is a <code>(string * t) list</code>; for these we need to convert <code>x</code> to the <code>Jq_comma</code> / <code>Jq_nil</code> representation above. But in the parser all we do is return a <code>Jq_Ant</code> containing the tag and body of the <code>ANTIQUOT</code> token. (We return it in a single string separated by <code>:</code> because only one string argument is provided in <code>ExAnt</code>.)</p> 
 
<p>It is the parser which controls where antiquotations are allowed, by providing a case for <code>ANTIQUOT</code> in a particular entry, and which tags are allowed in an entry. In this example we have only one entry, so we allow any supported antiquotation anywhere a JSON expression is allowed, but you can see in the OCaml parsers that the acceptable antiquotations can be context-sensitive, and the interpretation of the same antiquotation can vary according to the context (e.g. different conversions may be needed).</p> 
 
<p>For arrays and objects, we parse <code>SELF</code> in place of the list. The cases for <code>Jq_comma</code> and <code>Jq_nil</code> produce the tree representation, and the case for <code>Jq_colon</code> allows name/value pairs. Recall that a token or keyword is preferred over the empty string, so the <code>Jq_nil</code> case matches only when none of the others do. In particular, the quotation <code>&lt;:json&lt; &gt;&gt;</code> parses to <code>Jq_nil</code>.</p> 
 
<p>We can see that not only is the AST rather free, but so is the parser: it will parse strings which are not well-formed JSON, like <code>&lt;:json&lt; 1, 2 &gt;&gt;</code> or <code>&lt;json:&lt; &quot;foo&quot; : true &gt;&gt;</code>. We lose safety, since a mistake may produce an ill-formed AST, but gain convenience, since we may want to substitute these fragments in antiquotations. As an alternative, we could have a more restrictive parser (e.g. no commas allowed at the <code>json</code> entry), and provide different quotations for different contexts (e.g. <code>&lt;:json_list&lt; &gt;&gt;</code>, allowing commas) for use with antiquotations. For this small language I think it is not worth it.</p> 
<b>Expanding antiquotations</b> 
<p>To expand antiquotations, we take a pass over the OCaml AST we got from lifting the JSON AST; look for <code>ExAst</code> nodes; parse them as OCaml; then apply the appropriate conversion according to the antiquotation tag. To walk the AST we extend the <code>Ast.map</code> object (generated with the <code>Camlp4FoldGenerator</code> filter) so we don&rsquo;t need a bunch of boilerplate cases which return the AST unchanged. Here&rsquo;s the code:</p> 
<div class="highlight"><pre><code class="ocaml">  <span class="k">module</span> <span class="nc">AQ</span> <span class="o">=</span> <span class="nn">Syntax</span><span class="p">.</span><span class="nc">AntiquotSyntax</span> 
  
  <span class="k">let</span> <span class="n">destruct_aq</span> <span class="n">s</span> <span class="o">=</span> 
    <span class="k">let</span> <span class="n">pos</span> <span class="o">=</span> <span class="nn">String</span><span class="p">.</span><span class="n">index</span> <span class="n">s</span> <span class="sc">':'</span> <span class="k">in</span> 
    <span class="k">let</span> <span class="n">len</span> <span class="o">=</span> <span class="nn">String</span><span class="p">.</span><span class="n">length</span> <span class="n">s</span> <span class="k">in</span> 
    <span class="k">let</span> <span class="n">name</span> <span class="o">=</span> <span class="nn">String</span><span class="p">.</span><span class="n">sub</span> <span class="n">s</span> <span class="mi">0</span> <span class="n">pos</span> 
    <span class="ow">and</span> <span class="n">code</span> <span class="o">=</span> <span class="nn">String</span><span class="p">.</span><span class="n">sub</span> <span class="n">s</span> <span class="o">(</span><span class="n">pos</span> <span class="o">+</span> <span class="mi">1</span><span class="o">)</span> <span class="o">(</span><span class="n">len</span> <span class="o">-</span> <span class="n">pos</span> <span class="o">-</span> <span class="mi">1</span><span class="o">)</span> <span class="k">in</span> 
    <span class="n">name</span><span class="o">,</span> <span class="n">code</span> 
  
  <span class="k">let</span> <span class="n">aq_expander</span> <span class="o">=</span> 
  <span class="k">object</span> 
    <span class="k">inherit</span> <span class="nn">Ast</span><span class="p">.</span><span class="n">map</span> <span class="k">as</span> <span class="n">super</span> 
    <span class="k">method</span> <span class="n">expr</span> <span class="o">=</span> 
      <span class="k">function</span> 
        <span class="o">|</span> <span class="nn">Ast</span><span class="p">.</span><span class="nc">ExAnt</span> <span class="o">(_</span><span class="n">loc</span><span class="o">,</span> <span class="n">s</span><span class="o">)</span> <span class="o">-&gt;</span> 
            <span class="k">let</span> <span class="n">n</span><span class="o">,</span> <span class="n">c</span> <span class="o">=</span> <span class="n">destruct_aq</span> <span class="n">s</span> <span class="k">in</span> 
            <span class="k">let</span> <span class="n">e</span> <span class="o">=</span> <span class="nn">AQ</span><span class="p">.</span><span class="n">parse_expr</span> <span class="o">_</span><span class="n">loc</span> <span class="n">c</span> <span class="k">in</span> 
            <span class="k">begin</span> <span class="k">match</span> <span class="n">n</span> <span class="k">with</span> 
              <span class="o">|</span> <span class="s2">&quot;bool&quot;</span> <span class="o">-&gt;</span> <span class="o">&lt;:</span><span class="n">expr</span><span class="o">&lt;</span> <span class="nn">Jq_ast</span><span class="p">.</span><span class="nc">Jq_bool</span> <span class="o">$</span><span class="n">e</span><span class="o">$</span> <span class="o">&gt;&gt;</span> 
              <span class="o">|</span> <span class="s2">&quot;int&quot;</span> <span class="o">-&gt;</span> 
                  <span class="o">&lt;:</span><span class="n">expr</span><span class="o">&lt;</span> <span class="nn">Jq_ast</span><span class="p">.</span><span class="nc">Jq_number</span> <span class="o">(</span><span class="n">float_of_int</span> <span class="o">$</span><span class="n">e</span><span class="o">$)</span> <span class="o">&gt;&gt;</span> 
              <span class="o">|</span> <span class="s2">&quot;flo&quot;</span> <span class="o">-&gt;</span> <span class="o">&lt;:</span><span class="n">expr</span><span class="o">&lt;</span> <span class="nn">Jq_ast</span><span class="p">.</span><span class="nc">Jq_number</span> <span class="o">$</span><span class="n">e</span><span class="o">$</span> <span class="o">&gt;&gt;</span> 
              <span class="o">|</span> <span class="s2">&quot;str&quot;</span> <span class="o">-&gt;</span> <span class="o">&lt;:</span><span class="n">expr</span><span class="o">&lt;</span> <span class="nn">Jq_ast</span><span class="p">.</span><span class="nc">Jq_string</span> <span class="o">$</span><span class="n">e</span><span class="o">$</span> <span class="o">&gt;&gt;</span> 
              <span class="o">|</span> <span class="s2">&quot;list&quot;</span> <span class="o">-&gt;</span> <span class="o">&lt;:</span><span class="n">expr</span><span class="o">&lt;</span> <span class="nn">Jq_ast</span><span class="p">.</span><span class="n">t_of_list</span> <span class="o">$</span><span class="n">e</span><span class="o">$</span> <span class="o">&gt;&gt;</span> 
              <span class="o">|</span> <span class="s2">&quot;alist&quot;</span> <span class="o">-&gt;</span> 
                  <span class="o">&lt;:</span><span class="n">expr</span><span class="o">&lt;</span> 
                    <span class="nn">Jq_ast</span><span class="p">.</span><span class="n">t_of_list</span> 
                      <span class="o">(</span><span class="nn">List</span><span class="p">.</span><span class="n">map</span> 
                        <span class="o">(</span><span class="k">fun</span> <span class="o">(</span><span class="n">k</span><span class="o">,</span> <span class="n">v</span><span class="o">)</span> <span class="o">-&gt;</span> 
                          <span class="nn">Jq_ast</span><span class="p">.</span><span class="nc">Jq_colon</span> <span class="o">(</span><span class="nn">Jq_ast</span><span class="p">.</span><span class="nc">Jq_string</span> <span class="n">k</span><span class="o">,</span> <span class="n">v</span><span class="o">))</span> 
                        <span class="o">$</span><span class="n">e</span><span class="o">$)</span> 
                  <span class="o">&gt;&gt;</span> 
              <span class="o">|</span> <span class="o">_</span> <span class="o">-&gt;</span> <span class="n">e</span> 
            <span class="k">end</span> 
        <span class="o">|</span> <span class="n">e</span> <span class="o">-&gt;</span> <span class="n">super</span><span class="o">#</span><span class="n">expr</span> <span class="n">e</span> 
    <span class="k">method</span> <span class="n">patt</span> <span class="o">=</span> 
      <span class="k">function</span> 
        <span class="o">|</span> <span class="nn">Ast</span><span class="p">.</span><span class="nc">PaAnt</span> <span class="o">(_</span><span class="n">loc</span><span class="o">,</span> <span class="n">s</span><span class="o">)</span> <span class="o">-&gt;</span> 
            <span class="k">let</span> <span class="o">_,</span> <span class="n">c</span> <span class="o">=</span> <span class="n">destruct_aq</span> <span class="n">s</span> <span class="k">in</span> 
            <span class="nn">AQ</span><span class="p">.</span><span class="n">parse_patt</span> <span class="o">_</span><span class="n">loc</span> <span class="n">c</span> 
        <span class="o">|</span> <span class="n">p</span> <span class="o">-&gt;</span> <span class="n">super</span><span class="o">#</span><span class="n">patt</span> <span class="n">p</span> 
  <span class="k">end</span> 
</code></pre> 
</div> 
<p>When we find an antiquotation, we unpack the tag and contents (with <code>destruct_aq</code>), parse it using the host syntax (given by <code>Syntax.AntiquotSyntax</code> from <code>Camlp4.PreCast</code>, which might be either the original or revised syntax depending which modules are loaded), then insert conversions depending on the tag. Conversions don&rsquo;t make sense in a pattern context, so for patterns we just return the parsed antiquotation.</p> 
 
<p>Finally we hook into the quotation machinery, mostly as before:</p> 
<div class="highlight"><pre><code class="ocaml"><span class="k">let</span> <span class="n">parse_quot_string</span> <span class="n">loc</span> <span class="n">s</span> <span class="o">=</span> 
  <span class="k">let</span> <span class="n">q</span> <span class="o">=</span> <span class="o">!</span><span class="nn">Camlp4_config</span><span class="p">.</span><span class="n">antiquotations</span> <span class="k">in</span> 
  <span class="nn">Camlp4_config</span><span class="p">.</span><span class="n">antiquotations</span> <span class="o">:=</span> <span class="bp">true</span><span class="o">;</span> 
  <span class="k">let</span> <span class="n">res</span> <span class="o">=</span> <span class="nn">Jq_parser</span><span class="p">.</span><span class="nn">Gram</span><span class="p">.</span><span class="n">parse_string</span> <span class="n">json_eoi</span> <span class="n">loc</span> <span class="n">s</span> <span class="k">in</span> 
  <span class="nn">Camlp4_config</span><span class="p">.</span><span class="n">antiquotations</span> <span class="o">:=</span> <span class="n">q</span><span class="o">;</span> 
  <span class="n">res</span> 
 
<span class="k">let</span> <span class="n">expand_expr</span> <span class="n">loc</span> <span class="o">_</span> <span class="n">s</span> <span class="o">=</span> 
  <span class="k">let</span> <span class="n">ast</span> <span class="o">=</span> <span class="n">parse_quot_string</span> <span class="n">loc</span> <span class="n">s</span> <span class="k">in</span> 
  <span class="k">let</span> <span class="n">meta_ast</span> <span class="o">=</span> <span class="nn">Jq_ast</span><span class="p">.</span><span class="nn">MetaExpr</span><span class="p">.</span><span class="n">meta_t</span> <span class="n">loc</span> <span class="n">ast</span> <span class="k">in</span> 
  <span class="n">aq_expander</span><span class="o">#</span><span class="n">expr</span> <span class="n">meta_ast</span> 
 
<span class="o">;;</span> 
 
<span class="nn">Q</span><span class="p">.</span><span class="n">add</span> <span class="s2">&quot;json&quot;</span> <span class="nn">Q</span><span class="p">.</span><span class="nn">DynAst</span><span class="p">.</span><span class="n">expr_tag</span> <span class="n">expand_expr</span><span class="o">;</span> 
</code></pre> 
</div> 
<p>Before parsing a quotation we set a flag, which is checked by the lexer, to allow antiquotations; the flag is initially false, so antiquotations appearing outside a quotation won&rsquo;t be parsed. After lifting the JSON AST to an OCaml AST, we run the result through the antiquotation expander.</p> 
 
<p>For concreteness, let&rsquo;s follow the life of a quotation as it is parsed and expanded. Say we begin with</p> 
<div class="highlight"><pre><code class="ocaml">  <span class="o">&lt;:</span><span class="n">json</span><span class="o">&lt;</span> <span class="o">[</span> <span class="mi">1</span><span class="o">,</span> <span class="o">$</span><span class="kt">int</span><span class="o">:</span><span class="n">x</span><span class="o">$</span> <span class="o">]</span> <span class="o">&gt;&gt;</span> 
</code></pre> 
</div> 
<p>After parsing:</p> 
<div class="highlight"><pre><code class="ocaml">  <span class="nc">Jq_array</span> <span class="o">(</span><span class="nc">Jq_comma</span> <span class="o">(</span><span class="nc">Jq_number</span> <span class="mi">1</span><span class="o">.,</span> <span class="nc">Jq_Ant</span> <span class="o">(_</span><span class="n">loc</span><span class="o">,</span> <span class="s2">&quot;int:x&quot;</span><span class="o">)))</span> 
</code></pre> 
</div> 
<p>After lifting:</p> 
<div class="highlight"><pre><code class="ocaml">  <span class="o">&lt;:</span><span class="n">expr</span><span class="o">&lt;</span> 
    <span class="nc">Jq_array</span> <span class="o">(</span><span class="nc">Jq_comma</span> <span class="o">(</span><span class="nc">Jq_number</span> <span class="mi">1</span><span class="o">.,</span> <span class="o">$</span><span class="nc">ExAnt</span> <span class="o">(_</span><span class="n">loc</span><span class="o">,</span> <span class="s2">&quot;int:x&quot;</span><span class="o">)$))</span> 
  <span class="o">&gt;&gt;</span> 
</code></pre> 
</div> 
<p>After expanding:</p> 
<div class="highlight"><pre><code class="ocaml">  <span class="o">&lt;:</span><span class="n">expr</span><span class="o">&lt;</span> 
    <span class="nc">Jq_array</span> <span class="o">(</span><span class="nc">Jq_comma</span> <span class="o">(</span><span class="nc">Jq_number</span> <span class="mi">1</span><span class="o">.,</span> <span class="nc">Jq_number</span> <span class="o">(</span><span class="n">float_of_int</span> <span class="n">x</span><span class="o">)))</span> 
  <span class="o">&gt;&gt;</span> 
</code></pre> 
</div><b>Nested quotations</b> 
<p>Let&rsquo;s see that again with a nested quotation:</p> 
<div class="highlight"><pre><code class="ocaml">  <span class="o">&lt;:</span><span class="n">json</span><span class="o">&lt;</span> <span class="o">$&lt;:</span><span class="n">json</span><span class="o">&lt;</span> <span class="mi">1</span> <span class="o">&gt;&gt;$</span> <span class="o">&gt;&gt;</span> 
</code></pre> 
</div> 
<p>After parsing:</p> 
<div class="highlight"><pre><code class="ocaml">  <span class="nc">Jq_Ant</span> <span class="o">(_</span><span class="n">loc</span><span class="o">,</span> <span class="s2">&quot;&lt;:json&lt; 1 &gt;&gt;&quot;</span><span class="o">)</span> 
</code></pre> 
</div> 
<p>After lifting:</p> 
<div class="highlight"><pre><code class="ocaml">  <span class="nc">ExAnt</span> <span class="o">(_</span><span class="n">loc</span><span class="o">,</span> <span class="s2">&quot;&lt;:json&lt; 1 &gt;&gt;&quot;</span><span class="o">)</span> 
</code></pre> 
</div> 
<p>After expanding (during which we parse and expand <code>&quot;&lt;:json&lt; 1 &gt;&gt;&quot;</code> to <code>&lt;:expr&lt; Jq_number 1. &gt;&gt;</code>):</p> 
<div class="highlight"><pre><code class="ocaml">  <span class="o">&lt;:</span><span class="n">expr</span><span class="o">&lt;</span> <span class="nc">Jq_number</span> <span class="mi">1</span><span class="o">.</span> <span class="o">&gt;&gt;</span> 
</code></pre> 
</div> 
<p>A wise man <a href="http://www.cs.yale.edu/quotes.html">once said</a> &ldquo;The string is a stark data structure and everywhere it is passed there is much duplication of process.&rdquo; So it is with Camlp4 quotations: each nested quotation is re-parsed; each quotation implementation must deal with parsing host-language antiquotation strings; and the lexer for each implementation must lex antiquotations and nested quotations. (Since we used the default lexer we didn&rsquo;t have to worry about this, but see the next post.) It would be nice to have more support from Camlp4. On the other hand, while what happens at runtime seems baroque, the code above is relatively straightforward, and since we work with strings we can use any parser technology we like.</p> 
 
<p>It has not been much (marginal) trouble to handle quotations in pattern contexts, but they are not tremendously useful. The problem is that we normally don&rsquo;t care about the order of the fields in a JSON object, or if there are extra fields; we would like to write</p> 
<div class="highlight"><pre><code class="ocaml">  <span class="k">match</span> <span class="n">x</span> <span class="k">with</span> 
    <span class="o">|</span> <span class="o">&lt;:</span><span class="n">json</span><span class="o">&lt;</span> <span class="o">{</span> 
        <span class="s2">&quot;foo&quot;</span> <span class="o">:</span> <span class="o">$</span><span class="n">foo</span><span class="o">$</span> 
      <span class="o">}</span> <span class="o">&gt;&gt;</span> <span class="o">-&gt;</span> <span class="o">...</span> <span class="c">(* do something with foo *)</span> 
</code></pre> 
</div> 
<p>and have it work wherever the <code>foo</code> field is in the object. This is a more complicated job than just lifting the JSON AST. For an alternative approach to processing JSON using a list-comprehension syntax, see <a href="http://github.com/jaked/cufp-metaprogramming-tutorial/tree/master/ocaml/json_compr/">json_compr</a>, an example I wrote for the upcoming <a href="http://cufp.org/conference/sessions/2010/camlp4-and-template-haskell">metaprogramming tutorial at CUFP</a>. For a fancier JSON DSL (including the ability to induct a type description from a bunch of examples!), see Julien Verlauget&rsquo;s <a href="http://github.com/pika/jsonpat">jsonpat</a>. And for a framework to extend OCaml&rsquo;s pattern-matching syntax, see Jeremy Yallop&rsquo;s <a href="http://code.google.com/p/ocaml-patterns/">ocaml-patterns</a>.</p> 
 
<p>Next time we will see how to use a custom lexer with a Camlp4 grammar.</p> 
 
<p>(You can find the complete code for this example <a href="http://github.com/jaked/ambassadortothecomputers.blogspot.com/tree/master/_code/camlp4-implementing-antiquotations">here</a>.)</p>
