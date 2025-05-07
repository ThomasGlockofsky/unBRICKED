---
layout: post
title: Enable Extended Display on the LG V30
category: [LG V30]
---

## I DIDN'T BREAK IT
yet

# Android 9 on the LG V30 DOES NOT allow for extended display.
This means if you connect an external display to the type-c port, it will ONLY mirror the phone's internal display. This is fine for most  
people, but I like using all the screen space I have available. This phone in particular is not rooted, and the bootloader is LOCKED. It  
took me an embarrassingly long amount of time to bypass the FRP lock, and at this point I feel as if I've devoted too much time into getting
it into a usable state to risk bricking it. 

## Instructions:
First and foremost, this phone was originally on Verizon firmware, as it is a VS996, but in the process of bypassing the FRP, i crossed   
flashed T-Mobile firmware. It now believes it is a US998. Im not sure if this is necessary for this to work, but I've added to note just in case.

Step 1) Enable USB Debugging (duh), and connect to debugger (Win 11 dev edition and Android 15 on razr 2024 for me)  
Step 2) Run "adb shell dumpsys display". Note the display IDs and Layers. The Slimport should be display ID 1, layer0  
Step 3) Using scrcpy, run "scrcpy --list-display". The display ID for the Slimport should match from "dumpsys", if not, reboot and try again.  
Step 4) You will need to launch scrcpy on the second display (scrcpy --display-id 1). It should mirror the phone's screen.  
Step 5) In a seperate terminal, you need to launch an activity on the second display (using adb shell), using the display ID 1. It should fail. "adb shell am start --display 1 -n com.stremio.one/com.stremio.tv.MainActivity" for Stremio, but you can try any app.  
Step 6) Run "adb shell dumpsys display" again. This time the slimport should have a layer number other than 0, and the display ID should 
change to match it.  
Step 7) You should now be able to launch an activity successfully on the second display(using adb shell), using the new display ID. Adb shell am start --display 13 -n com.stremio.one/com.stremio.tv.MainActivity 

So far the display ID doesn't persist after reboot, although the ability to extend the display does, so you'll have to change the ID after
rebooting, but that should be it. Also you can close scrcpy now.

## I *just* got this working about 2 or 3 days ago, so I'm not really sure of the limits, or any bugs that may exist.

So far I've been using it to stream on my TV, while running separate apps on the phone screen. In order to regain focus on the external display, just tap the icon for whichever app is running. Again, so far I've only tested this with Stremio and Flauncher, but in theory it
should work with any app. Also things like volume and power menus won't show on the external display at the moment, although I'm sure it  could with some more fuckery. Gotta fuck around to find out.

## I'll update this post as I as continue to work on this project.
~~I rebooted my compouter last night and Windows doesn't save command history, but once I figure out the proper commands I'll add them.~~  
The command has been added to the instructions, you can replace Stremio with any other app and it should work. 

{% raw %}
<div>
  <button onclick="vote('like')">Like (<span id="like-count">0</span>)</button>
  <button onclick="vote('dislike')">Dislike (<span id="dislike-count">0</span>)</button>
</div>

<script type="module">
  import { createClient } from 'https://cdn.jsdelivr.net/npm/@supabase/supabase-js/+esm'

  const supabase = createClient('https://motdgqbhzfurezxmgoxs.supabase.co', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6Im1vdGRncWJoemZ1cmV6eG1nb3hzIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NDY2Mjc0NjIsImV4cCI6MjA2MjIwMzQ2Mn0.GMeVFEWbdzl3HxvRJerSCQA4Tg9tDrey9ILESrHTVNQ')
  const page = 'EnablingExtendedDisplay.md';

console.log('Page ID:',page);

  async function updateCounts() {
    const { data, error } = await supabase
      .from('votes')
      .select('*')
      .eq('page', page)
      .single()

    if (data) {
      document.getElementById('like-count').textContent = data.likes
      document.getElementById('dislike-count').textContent = data.dislikes
    }
  }

 window.vote = async function (type) {
    const { data, error } = await supabase
      .from('votes')
      .select('*')
      .eq('page', page)
      .single()


console.log('Page:', page);
console.log('Data:', data);
console.error('Error:', error);
    
    if (data) {
      const updated = {
        likes: type === 'like' ? data.likes + 1 : data.likes,
        dislikes: type === 'dislike' ? data.dislikes + 1 : data.dislikes
      }

      await supabase
        .from('votes')
        .update(updated)
        .eq('page', page)
    } else {
      await supabase
        .from('votes')
        .insert([{ page, likes: type === 'like' ? 1 : 0, dislikes: type === 'dislike' ? 1 : 0 }])
    }

    updateCounts()
  }

  updateCounts()

  // Get user IP for uniqueness (can be improved later)
async function getIP() {
  try {
    const res = await fetch("https://api.ipify.org?format=json");
    const json = await res.json();
    return json.ip;
  } catch {
    return null;
  }
}

async function recordView() {
  const ip = await getIP();

  const { data, error } = await supabase
    .from('views')
    .select('*')
    .eq('page', page)
    .single();

  if (error && error.code !== 'PGRST116') {
    console.error('View fetch error:', error);
    return;
  }

  if (!data) {
    // First time this page is viewed
    await supabase.from('views').insert({
      page: page,
      total_views: 1,
      unique_views: ip ? 1 : 0,
      last_ip: ip
    });
  } else {
    const isUnique = ip && data.last_ip !== ip;
    await supabase.from('views').update({
      total_views: data.total_views + 1,
      unique_views: data.unique_views + (isUnique ? 1 : 0),
      last_ip: ip
    }).eq('page', page);
  }

  // Display the result
  const result = await supabase
    .from('views')
    .select('total_views, unique_views')
    .eq('page', page)
    .single();

  if (result.data) {
    document.getElementById('views').innerText =
      `Views: ${result.data.total_views} | Unique: ${result.data.unique_views}`;
  }
}

recordView();

  async function loadComments() {
  const { data, error } = await supabase
    .from('comments')
    .select('username, body, created_at')
    .eq('page', page)
    .order('created_at', { ascending: false });

  if (!error) {
    const container = document.getElementById('comments');
    container.innerHTML = '';
    data.forEach(c => {
      const el = document.createElement('div');
      el.innerHTML = `<strong>${c.username}</strong> <small>${new Date(c.created_at).toLocaleString()}</small><p>${c.body}</p><hr>`;
      container.appendChild(el);
    });
  }
}

document.getElementById('comment-form').addEventListener('submit', async (e) => {
  e.preventDefault();
  const username = document.getElementById('username').value.trim();
  const body = document.getElementById('comment-body').value.trim();

  if (username && body) {
    await supabase.from('comments').insert([{ page, username, body }]);
    document.getElementById('comment-body').value = '';
    loadComments();
  }
});

loadComments();
  
</script>
{% endraw %}
<p id="views">Loading views...</p>

<div id="comment-section">
  <h3>Comments</h3>
  <div id="comments"></div>

  <form id="comment-form">
    <input type="text" id="username" placeholder="Your name" required style="display:block; margin-bottom:5px;">
    <textarea id="comment-body" placeholder="Write a comment..." required style="display:block; width:100%; height:80px;"></textarea>
    <button type="submit">Post Comment</button>
  </form>
</div>
