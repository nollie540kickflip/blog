---
layout: page
title: 記事の検索
permalink: /search/
---

<style>
  #search-input {
    width: 100%;
    padding: 10px;
    font-size: 16px;
    border: 1px solid #ccc;
    border-radius: 4px;
    margin-bottom: 20px;
  }
  #search-results {
    list-style-type: none;
    padding: 0;
  }
  #search-results li {
    margin-bottom: 15px;
    border-bottom: 1px solid #eee;
    padding-bottom: 10px;
  }
  #search-results h3 {
    margin: 0 0 5px 0;
  }
  #search-results a {
    text-decoration: none;
    color: #2a7ae2;
  }
  #search-results p {
    margin: 0;
    color: #666;
    font-size: 14px;
  }
</style>

<input type="text" id="search-input" placeholder="キーワードを入力してください..." autofocus>

<ul id="search-results"></ul>

<script>
  // JSONデータを取得する
  fetch('{{ site.baseurl }}/search.json')
    .then(response => response.json())
    .then(posts => {
      const input = document.getElementById('search-input');
      const results = document.getElementById('search-results');

      // 入力があるたびに検索を実行
      input.addEventListener('input', (e) => {
        const query = e.target.value.toLowerCase();
        results.innerHTML = ''; // 結果をクリア

        if (query.length === 0) {
          return;
        }

        // タイトルか本文にキーワードが含まれている記事を抽出
        const matched = posts.filter(post => 
          post.title.toLowerCase().includes(query) || 
          post.content.toLowerCase().includes(query)
        );

        if (matched.length === 0) {
          results.innerHTML = '<li>見つかりませんでした。</li>';
          return;
        }

        // 検索結果を表示
        matched.forEach(post => {
          const li = document.createElement('li');
          // 本文の一部を抜粋（最大100文字程度）
          const snippet = post.content.length > 100 
            ? post.content.substring(0, 100) + '...' 
            : post.content;
            
          li.innerHTML = `
            <h3><a href="${post.url}">${post.title}</a></h3>
            <p>${post.date} - ${snippet}</p>
          `;
          results.appendChild(li);
        });
      });
    });
</script>
