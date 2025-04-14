<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Drevanox</title>
  <style>
    @import url('https://fonts.googleapis.com/css2?family=Cinzel:wght@700&family=Roboto+Slab&display=swap');

    body {
      margin: 0;
      font-family: 'Roboto Slab', serif;
      color: #fff;
      background-color: #000;
      background-image: url('https://www.transparenttextures.com/patterns/black-linen.png');
      animation: fadeIn 2s ease-in;
    }

    @keyframes fadeIn {
      from { opacity: 0; }
      to { opacity: 1; }
    }

    header {
      padding: 2rem;
      display: flex;
      align-items: center;
      gap: 1rem;
      background: radial-gradient(circle at center, #111 0%, #000 100%);
      border-bottom: 2px solid crimson;
    }
    header img {
      height: 120px;
      animation: zoomIn 1s ease-in-out;
    }
    @keyframes zoomIn {
      from { transform: scale(0.5); opacity: 0; }
      to { transform: scale(1); opacity: 1; }
    }
    header h1 {
      font-family: 'Cinzel', serif;
      font-size: 3rem;
      color: crimson;
      margin: 0;
    }
    nav {
      background-color: #1a1a1a;
      display: flex;
      justify-content: space-around;
      border-top: 2px solid crimson;
      border-bottom: 2px solid crimson;
    }
    nav a {
      color: crimson;
      text-decoration: none;
      font-weight: bold;
      padding: 0.5rem 1rem;
      transition: all 0.3s ease-in-out;
    }
    nav a:hover {
      background-color: crimson;
      color: #000;
    }
    section {
      min-height: 100vh;
      border-bottom: 1px solid #333;
      background-color: rgba(0, 0, 0, 0.85);
      padding: 2rem;
      animation: fadeIn 1.5s ease-in;
    }
    h2 {
      color: crimson;
      font-family: 'Cinzel', serif;
    }
    blockquote {
      font-style: italic;
      border-left: 4px solid crimson;
      padding-left: 1rem;
      color: #ccc;
    }
    footer {
      background-color: #111;
      text-align: center;
      padding: 2rem;
      border-top: 1px solid #333;
    }
    .visual-section {
      display: flex;
      flex-wrap: wrap;
      gap: 1rem;
    }
    .visual-section img {
      width: 100%;
      max-width: 400px;
      height: auto;
      border: 2px solid crimson;
      border-radius: 10px;
      transition: transform 0.3s ease-in-out;
    }
    .visual-section img:hover {
      transform: scale(1.05);
    }
    .quote-text {
      font-family: 'Cinzel', serif;
      font-weight: bold;
      text-align: center;
      font-size: 2rem;
      margin-top: 30vh;
      animation: slideIn 2s ease-in-out;
    }
    .quote-author {
      text-align: center;
      font-size: 1.2rem;
      margin-top: 1rem;
      color: crimson;
    }
    @keyframes slideIn {
      from { transform: translateY(-50px); opacity: 0; }
      to { transform: translateY(0); opacity: 1; }
    }
    form {
      display: flex;
      flex-direction: column;
      max-width: 500px;
      margin: 2rem auto;
      gap: 1rem;
    }
    form input, form textarea {
      padding: 0.75rem;
      border-radius: 5px;
      border: 1px solid #444;
      font-family: 'Roboto Slab', serif;
    }
    form button {
      background-color: crimson;
      color: #000;
      border: none;
      padding: 1rem;
      font-weight: bold;
      cursor: pointer;
      transition: background-color 0.3s ease;
    }
    form button:hover {
      background-color: #a00000;
    }
  </style>
</head>
<body>
  <header>
    <img src="DREVANOX LOGO.png" alt="Drevanox Logo" />
    <div>
      <h1>DREVANOX</h1>
      <p style="color: #ccc; font-family: 'Roboto Slab', serif;">Forged in Eternity. Focused on You.</p>
    </div>
  </header>

  <nav>
    <a href="#started">How We Started</a>
    <a href="#special">What's Special</a>
    <a href="#choose">Why Choose Us</a>
    <a href="#clients">Clients' Perspective</a>
    <a href="#quote">Quote</a>
    <a href="#testimonials">Testimonials</a>
    <a href="#contact">Contact</a>
  </nav>

  <section id="started">
    <h2>How We Started</h2>
    <p>Drevanox was born from ambition, built on grit, and fueled by the fire of innovation...</p>
    <div class="visual-section">
      <img src="https://images.unsplash.com/photo-1535223289827-42f1e9919769" alt="Journey" />
    </div>
  </section>

  <section id="special">
    <h2>What's Special About Us</h2>
    <p>We mix mythic energy with modern execution. No fluff, just results that speak for themselves.</p>
    <div class="visual-section">
      <img src="https://images.unsplash.com/photo-1556742400-b5c05c9b84b8" alt="Special Approach" />
    </div>
  </section>

  <section id="choose">
    <h2>Why Should People Choose Us?</h2>
    <p>Because we're more than a service — we're an experience crafted to endure.</p>
    <div class="visual-section">
      <img src="https://images.unsplash.com/photo-1605810230434-7631cf222b5c" alt="Why Us" />
    </div>
  </section>

  <section id="clients">
    <h2>Why Our Clients Agree to Work for Us</h2>
    <p>They don’t just stay — they advocate. They believe in what we build together.</p>
    <div class="visual-section">
      <img src="https://images.unsplash.com/photo-1595152772835-219674b2a8a6" alt="Happy Clients" />
    </div>
  </section>

  <section id="quote">
    <div class="quote-text">
      "The best business strategy is a satisfied customer."
    </div>
    <div class="quote-author">
      — CEO, Drevanox
    </div>
  </section>

  <section id="testimonials">
    <h2>Testimonials</h2>
    <p>“Drevanox changed the way we do business.” — A Satisfied Partner</p>
    <p>“Unmatched clarity and results.” — Another Happy Client</p>
    <div class="visual-section">
      <img src="https://images.unsplash.com/photo-1521737604893-d14cc237f11d" alt="Client Testimonials" />
    </div>
  </section>

  <section id="contact">
    <h2>Contact Us</h2>
    <form>
      <input type="text" placeholder="First Name" required />
      <input type="text" placeholder="Last Name" required />
      <input type="text" placeholder="Skype ID" required />
      <input type="text" placeholder="Contact Number" required />
      <input type="email" placeholder="Email" required />
      <textarea placeholder="Your Comment" rows="4"></textarea>
      <button type="submit">Submit</button>
    </form>
  </section>

  <footer>
    <p>&copy; 2025 Drevanox. All rights reserved.</p>
  </footer>
</body>
</html>
