---
layout: post
title: "Autocomplete with Elasticsearch and Tire"
date: 2013-06-16 14:09
comments: true
author: Ilan 
categories: [Technical, Rails]
---

We’ve recently seen a need to introduce an autocomplete feature to [Tipter](http://tipter.com). [Tipter](http://tipter.com) 
allows its users to search for Trips (a.k.a Travel Blogs) and Tips (the building blocks of Trips). I’ve spent couple of days on this feature, integrating multiple resources until I got the desired result.

This list of resources includes: [this great blog post](http://jontai.me/blog/2013/02/adding-autocomplete-to-an-elasticsearch-search-application/), which refers to an excellent Stack Overflow [answer](http://stackoverflow.com/questions/9421358/filename-search-with-elasticsearch/9432450#9432450), and this [issue](https://github.com/karmi/tire/issues/101) on github which explains how to use multifield in Tire. It also includes this [blog post](http://masonoise.wordpress.com/2012/08/11/elasticsearch-with-rails-and-tire/) 
which I only found at later stages, definitely could have saved me some time. And of course [Elasticsearch](http://www.elasticsearch.org/guide/) guide, [Tire](https://github.com/karmi/tire) documentation, and [Railscast](http://railscasts.com/episodes/306-elasticsearch-part-1). 

Let’s start by looking at some code. I’ve listed the relevant code snippets of our trip model. Each trip `has_many` countries which `has_one` sovereign model that hold the country’s name. Our target (for this post at least…) is to be able to perform autocomplete on the country name, and offer the user relevant travel blogs that include that country.

    class Trip < ActiveRecord::Base
      has_many :countries, dependent: :destroy
  
      def countries_names
        sovereign_names = self.countries.map { |c| c.sovereign.nil? ? "" : c.sovereign.name }.join(", ")
        sovereign_names
      end
  
  
      include Tire::Model::Search
      include Tire::Model::Callbacks
    
      settings :analysis => { 
                 :filter => {
                   :trip_ngram  => {
                     "type"     => "edgeNGram",
                     "max_gram" => 15,
                     "min_gram" => 2 }
                 },
                 :analyzer => {
                   :index_ngram_analyzer => {
                     "type" => "custom",
                     "tokenizer" => "standard",
                     "filter" => [ "standard", "lowercase", "trip_ngram" ] 
                   }, 
                   :search_ngram_analyzer => {
                     "type" => "custom",
                     "tokenizer" => "standard",
                     "filter" => [ "standard", "lowercase"] 
                   }, 
                 }     
              } 
    

      mapping do 
       indexes :countries_names, :type => 'multi_field', :fields => {
          :countries_names => { :type => "string"},
          :"countries_names.autocomplete" => { :search_analyzer => "search_ngram_analyzer", :index_analyzer => "index_ngram_analyzer", :type => "string"}
       }
      end

      def to_indexed_json 
        to_json(methods: [:countries_names])
      end

      def self.regular_search(params) 
        tire.search(load: true) do
           query {string 'countries_names:' + params }
        end
      end

   
      def self.autocomplete(params) 
        tire.search(load: true) do
           query {string 'countries_names.autocomplete:' + params }
        end
      end

So, for a start we'll index the `countries_names` using the `mapping` block code. We will use the Elasticsearch `multi_field` type to define "regular" `countries_names` search, and autocomplete one. 
This will allow us to declare two different methods able to be used based on our needs. For the `countries_names:autocomplete` variant, we declare two types of analyzers `:search_analyzer` and `:index_analyzer`.  

A great overview of analyzers' structure and how they work in Elasticsearch can be found [here](http://jontai.me/blog/2013/02/adding-autocomplete-to-an-elasticsearch-search-application/), and I don't want to repeat that (and probably introduce errors... :-) ). However, some of the modifications I did were to use Edge NGram token filter so that n-grams will only be generated from the beginning of the word. 

In addition I used different filters for search and indexing where the search filter doesn’t use the Edge NGram filter. 

My initial version of the mapping used a single analyzer:

     mapping do 
       indexes :countries_names, :type => 'multi_field', :fields => {
          :countries_names => { :type => "string"},
          :"countries_names.autocomplete" => { :analyzer => "index_ngram_analyzer", :type => "string"}
       }
     end

While playing with several search queries on the console, I ran this query:

    1.9.3-p327 :159 > Trip.autocomplete("mar").map { |t| t.countries_names }
     ....
     => ["Malaysia", "Malaysia"] 

The reason it found Malaysia is because of the index containing the following tokens:

[ma], [mal], [mala], [malay], [malays], [malaysi], [malaysia]

Notice I used `min_gram` of two in order to reduces false positives. Using the same token filter, the search query translates to:

[ma], [mar]

So there's a match on the "ma". This can be avoided by using different search analyzer as in the first code snippet. Running the same query gets the expected result:

    1.9.3-p327 :163 > Trip.autocomplete("mar").map { |t| t.countries_names }
     => [] 

    1.9.3-p327 :164 > Trip.autocomplete("ma").map { |t| t.countries_names }
     ...
     => ["Malaysia", "Malaysia"] 

    1.9.3-p327 :167 > Trip.regular_search("ma").map { |t| t.countries_names }
    => [] 

There you go. I hope this will save some of you the time I spent on wiring everything together.


