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
      background-color: #000;
      color: #fff;
      background-image: url('https://www.transparenttextures.com/patterns/dark-fish-skin.png');
    }
    header, nav, section, footer {
      padding: 2rem;
    }
    header {
      display: flex;
      align-items: center;
      gap: 1rem;
      background: radial-gradient(circle at center, #111 0%, #000 100%);
      border-bottom: 2px solid crimson;
    }
    header img {
      height: 120px;
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
    <a href="#main">Home</a>
    <a href="#started">How We Started</a>
    <a href="#special">What's Special</a>
    <a href="#choose">Why Choose Us</a>
    <a href="#clients">Clients' Perspective</a>
    <a href="#quote">Quote</a>
    <a href="#testimonials">Testimonials</a>
    <a href="#contact">Contact</a>
  </nav>

  <section id="main">
    <h2>Main Page</h2>
    <p>Welcome to Drevanox. We are the forge behind the future.</p>
    <div class="visual-section">
      <img src="https://images.unsplash.com/photo-1549921296-3a3b5df2bb99" alt="Futuristic Design" />
      <img src="https://images.unsplash.com/photo-1607746882042-944635dfe10e" alt="Mystical Technology" />
    </div>
  </section>

  <section id="started">
    <h2>How We Started</h2>
    <p>Drevanox was born from ambition, built on grit, and fueled by the fire of innovation...</p>
  </section>

  <section id="special">
    <h2>What's Special About Us</h2>
    <p>We mix mythic energy with modern execution. No fluff, just results that speak for themselves.</p>
  </section>

  <section id="choose">
    <h2>Why Should People Choose Us?</h2>
    <p>Because we're more than a service — we're an experience crafted to endure.</p>
  </section>

  <section id="clients">
    <h2>Why Our Clients Agree to Work for Us</h2>
    <p>They don’t just stay — they advocate. They believe in what we build together.</p>
  </section>

  <section id="quote">
    <h2>Featured Quote</h2>
    <blockquote>
      "Here you’ll add a powerful quote that represents Drevanox’s philosophy."
    </blockquote>
  </section>

  <section id="testimonials">
    <h2>Testimonials</h2>
    <p>“Drevanox changed the way we do business.” — A Satisfied Partner</p>
    <p>“Unmatched clarity and results.” — Another Happy Client</p>
  </section>

  <section id="contact">
    <h2>Contact Us</h2>
    <p>Email: contact@drevanox.com</p>
    <p>Phone: +1 234 567 8900</p>
    <p>Location: Everywhere innovation burns bright.</p>
  </section>

  <footer>
    <p>&copy; 2025 Drevanox. All rights reserved.</p>
  </footer>
</body>
</html>
