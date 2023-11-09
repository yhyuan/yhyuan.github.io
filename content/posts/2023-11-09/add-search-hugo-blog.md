---
title: "Add Search to Hugo Blog"
date: 2023-11-09T09:23:05-05:00
draft: false
---

Based on [this tutorial](https://gist.github.com/ludard/cac7ec7f9c1b167dbcb8b87210c72956)
## Add the file shown here in root directory or under themes/<themeName>
### content/search.md
```
---
title: "Search Results"
sitemap:
  priority : 0.1
layout: "search"
---


This file exists solely to respond to /search URL with the related `search` layout template.

No content shown here is rendered, all content is based in the template layouts/page/search.html

Setting a very low sitemap priority will tell search engines this is not important content.

This implementation uses Fusejs, jquery and mark.js


## Initial setup

Search  depends on additional output content type of JSON in config.toml
\```
[outputs]
  home = ["HTML", "JSON"]
\```

## Searching additional fileds

To search additional fields defined in front matter, you must add it in 2 places.

### Edit layouts/_default/index.JSON
This exposes the values in /index.json
i.e. add `category`
\```
...
  "contents":{{ .Content | plainify | jsonify }}
  {{ if .Params.tags }},
  "tags":{{ .Params.tags | jsonify }}{{end}},
  "categories" : {{ .Params.categories | jsonify }},
...
\```

### Edit fuse.js options to Search
`static/js/search.js`
\```
keys: [
  "title",
  "contents",
  "tags",
  "categories"
]
\```

```

### layouts/_default/search.html
```
{{ define "footerfiles" }}
<script src="https://code.jquery.com/jquery-3.3.1.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/fuse.js/3.2.0/fuse.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/mark.js/8.11.1/jquery.mark.min.js"></script>
<script src="{{ "js/search.js" | absURL }}"></script>
{{ end }}
{{ define "main" }}
<section class="resume-section p-3 p-lg-5 d-flex flex-column">
  <div class="my-auto" >
    <form action="{{ "search" | absURL }}">
      <input id="search-query" name="s"/>
    </form>
    <div id="search-results">
     <h3>Matching pages</h3>
    </div>
  </div>
</section>
<!-- this template is sucked in by search.js and appended to the search-results div above. So editing here will adjust style -->
<script id="search-result-template" type="text/x-js-template">
    <div id="summary-${key}">
      <h4><a href="${link}">${title}</a></h4>
      <p>${snippet}</p>
      ${ isset tags }<p>Tags: ${tags}</p>${ end }
      ${ isset categories }<p>Categories: ${categories}</p>${ end }
    </div>
</script>
{{ end }}
```

### static/js/search.js
```
summaryInclude=60;
var fuseOptions = {
  shouldSort: true,
  includeMatches: true,
  threshold: 0.0,
  tokenize:true,
  location: 0,
  distance: 100,
  maxPatternLength: 32,
  minMatchCharLength: 1,
  keys: [
    {name:"title",weight:0.8},
    {name:"contents",weight:0.5},
    {name:"tags",weight:0.3},
    {name:"categories",weight:0.3}
  ]
};


var searchQuery = param("s");
if(searchQuery){
  $("#search-query").val(searchQuery);
  executeSearch(searchQuery);
}else {
  $('#search-results').append("<p>Please enter a word or phrase above</p>");
}



function executeSearch(searchQuery){
  $.getJSON( "/index.json", function( data ) {
    var pages = data;
    var fuse = new Fuse(pages, fuseOptions);
    var result = fuse.search(searchQuery);
    console.log({"matches":result});
    if(result.length > 0){
      populateResults(result);
    }else{
      $('#search-results').append("<p>No matches found</p>");
    }
  });
}

function populateResults(result){
  $.each(result,function(key,value){
    var contents= value.item.contents;
    var snippet = "";
    var snippetHighlights=[];
    var tags =[];
    if( fuseOptions.tokenize ){
      snippetHighlights.push(searchQuery);
    }else{
      $.each(value.matches,function(matchKey,mvalue){
        if(mvalue.key == "tags" || mvalue.key == "categories" ){
          snippetHighlights.push(mvalue.value);
        }else if(mvalue.key == "contents"){
          start = mvalue.indices[0][0]-summaryInclude>0?mvalue.indices[0][0]-summaryInclude:0;
          end = mvalue.indices[0][1]+summaryInclude<contents.length?mvalue.indices[0][1]+summaryInclude:contents.length;
          snippet += contents.substring(start,end);
          snippetHighlights.push(mvalue.value.substring(mvalue.indices[0][0],mvalue.indices[0][1]-mvalue.indices[0][0]+1));
        }
      });
    }

    if(snippet.length<1){
      snippet += contents.substring(0,summaryInclude*2);
    }
    //pull template from hugo templarte definition
    var templateDefinition = $('#search-result-template').html();
    //replace values
    var output = render(templateDefinition,{key:key,title:value.item.title,link:value.item.permalink,tags:value.item.tags,categories:value.item.categories,snippet:snippet});
    $('#search-results').append(output);

    $.each(snippetHighlights,function(snipkey,snipvalue){
      $("#summary-"+key).mark(snipvalue);
    });

  });
}

function param(name) {
    return decodeURIComponent((location.search.split(name + '=')[1] || '').split('&')[0]).replace(/\+/g, ' ');
}

function render(templateString, data) {
  var conditionalMatches,conditionalPattern,copy;
  conditionalPattern = /\$\{\s*isset ([a-zA-Z]*) \s*\}(.*)\$\{\s*end\s*}/g;
  //since loop below depends on re.lastInxdex, we use a copy to capture any manipulations whilst inside the loop
  copy = templateString;
  while ((conditionalMatches = conditionalPattern.exec(templateString)) !== null) {
    if(data[conditionalMatches[1]]){
      //valid key, remove conditionals, leave contents.
      copy = copy.replace(conditionalMatches[0],conditionalMatches[2]);
    }else{
      //not valid, remove entire section
      copy = copy.replace(conditionalMatches[0],'');
    }
  }
  templateString = copy;
  //now any conditionals removed we can do simple substitution
  var key, find, re;
  for (key in data) {
    find = '\\$\\{\\s*' + key + '\\s*\\}';
    re = new RegExp(find, 'g');
    templateString = templateString.replace(re, data[key]);
  }
  return templateString;
}
```

### layouts/_default/index.json
```
{{- $.Scratch.Add "index" slice -}}
{{- range .Site.RegularPages -}}
    {{- $.Scratch.Add "index" (dict "title" .Title "tags" .Params.tags "categories" .Params.categories "contents" .Plain "permalink" .Permalink) -}}
{{- end -}}
{{- $.Scratch.Get "index" | jsonify -}}
```

### Config.toml
```
[outputs]
  home = ["HTML", "RSS", "JSON"]
```
or 
```
outputs:
- html
- rss
- json
```

### layouts/partials/footer_custom.html
add 
```
<script src="https://cdn.jsdelivr.net/npm/fuse.js@7.0.0"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/mark.js/8.11.1/jquery.mark.min.js"></script>
<script src="{{ "js/search.js" | absURL }}"></script>
```