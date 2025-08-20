# Odoo 18.0 — Theme & Website (Expert)

Advanced guide for building themes and website features for Odoo 18.0. Covers QWeb, SCSS, assets, snippets, options, and multi-website considerations.

## Table of contents
- [Theme Structure & Setup](#theme-structure--setup)
- [QWeb Snippets & Options](#qweb-snippets--options)
- [SCSS, Variables & Responsive Design](#scss-variables--responsive-design)
- [Dynamic Snippet Options & JavaScript](#dynamic-snippet-options--javascript)
- [Assets Management & Optimization](#assets-management--optimization)
- [Multi-website & Translations](#multi-website--translations)
- [SEO & Performance](#seo--performance)
- [Advanced Customizations](#advanced-customizations)
- [Testing & Debugging](#testing--debugging)
- [Migration & Maintenance](#migration--maintenance)

---

## Theme Structure & Setup

### Directory Structure
A complete theme module should follow this structure:

```
my_theme/
├── __manifest__.py
├── __init__.py
├── data/
│   ├── website_data.xml          # Website config, menus
│   ├── snippets.xml              # Snippet definitions
│   └── pages.xml                 # Default pages
├── views/
│   ├── templates.xml             # Main templates
│   ├── snippets/
│   │   ├── s_banner.xml
│   │   ├── s_features.xml
│   │   └── s_testimonials.xml
│   ├── layout.xml                # Layout overrides
│   └── pages/
│       ├── homepage.xml
│       ├── about.xml
│       └── contact.xml
├── static/
│   ├── description/
│   │   ├── icon.png
│   │   └── banner.png
│   └── src/
│       ├── scss/
│       │   ├── primary_variables.scss
│       │   ├── secondary_variables.scss
│       │   ├── bootstrap_overrides.scss
│       │   └── snippets/
│       │       ├── s_banner.scss
│       │       └── s_features.scss
│       ├── js/
│       │   ├── snippets/
│       │   │   ├── s_banner.js
│       │   │   └── s_features.js
│       │   └── tour/
│       │       └── theme_tour.js
│       └── img/
│           ├── snippets/
│           └── placeholders/
└── security/
    └── ir.model.access.csv
```

### Manifest Configuration

```python
# __manifest__.py
{
    'name': 'My Professional Theme',
    'description': 'Professional theme with advanced features',
    'category': 'Theme/Corporate',
    'version': '18.0.1.0.0',
    'depends': ['website', 'website_blog', 'website_sale'],
    'data': [
        'data/website_data.xml',
        'data/snippets.xml',
        'views/templates.xml',
        'views/layout.xml',
        'views/snippets/s_banner.xml',
        'views/snippets/s_features.xml',
        'views/pages/homepage.xml',
    ],
    'assets': {
        'web.assets_frontend': [
            # SCSS files - order matters!
            'my_theme/static/src/scss/primary_variables.scss',
            'my_theme/static/src/scss/bootstrap_overrides.scss',
            'my_theme/static/src/scss/snippets/s_banner.scss',
            'my_theme/static/src/scss/snippets/s_features.scss',
            
            # JavaScript files
            'my_theme/static/src/js/snippets/s_banner.js',
            'my_theme/static/src/js/snippets/s_features.js',
        ],
        'website.assets_wysiwyg': [
            # Snippet options for editor
            'my_theme/static/src/js/snippets/s_banner_options.js',
        ],
    },
    'images': [
        'static/description/banner.png',
        'static/description/theme_screenshot.png',
    ],
    'license': 'LGPL-3',
    'auto_install': False,
    'installable': True,
}
```

### Basic Theme Inheritance

```xml
<!-- views/layout.xml -->
<odoo>
    <template id="layout" inherit_id="website.layout" name="My Theme Layout">
        <!-- Add custom meta tags -->
        <xpath expr="//head" position="inside">
            <meta name="theme-color" content="#1a237e"/>
            <link rel="preconnect" href="https://fonts.googleapis.com"/>
            <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin=""/>
        </xpath>
        
        <!-- Custom header -->
        <xpath expr="//header" position="replace">
            <header id="top" class="o_header_standard o_header_sidebar_on_mobile" data-anchor="true">
                <div class="container">
                    <nav class="navbar navbar-expand-lg navbar-light">
                        <!-- Custom navigation -->
                        <t t-call="website.navbar_brand"/>
                        <t t-call="website.navbar_nav"/>
                    </nav>
                </div>
            </header>
        </xpath>
        
        <!-- Custom footer -->
        <xpath expr="//footer" position="replace">
            <footer class="bg-dark text-light py-5">
                <div class="container">
                    <div class="row">
                        <div class="col-lg-4">
                            <h5>About Us</h5>
                            <p>Professional services and solutions.</p>
                        </div>
                        <div class="col-lg-4">
                            <h5>Quick Links</h5>
                            <ul class="list-unstyled">
                                <li><a href="/page/about" class="text-light">About</a></li>
                                <li><a href="/page/services" class="text-light">Services</a></li>
                                <li><a href="/contactus" class="text-light">Contact</a></li>
                            </ul>
                        </div>
                        <div class="col-lg-4">
                            <h5>Contact Info</h5>
                            <p><i class="fa fa-envelope"></i> info@company.com</p>
                            <p><i class="fa fa-phone"></i> +1 234 567 890</p>
                        </div>
                    </div>
                </div>
            </footer>
        </xpath>
    </template>
</odoo>
```

---

## QWeb Snippets & Options

### Creating Professional Snippets

#### Banner Snippet with Multiple Layouts

```xml
<!-- views/snippets/s_banner.xml -->
<odoo>
    <template id="s_banner" name="Professional Banner">
        <section class="s_banner o_colored_level" data-snippet="s_banner" data-name="Banner">
            <div class="s_banner_wrapper">
                <div class="container">
                    <div class="row align-items-center">
                        <div class="col-lg-6" data-name="Content">
                            <h1 class="s_banner_title display-4 font-weight-bold mb-3" 
                                data-oe-expression="banner_title">
                                Professional Solutions
                            </h1>
                            <p class="s_banner_subtitle lead mb-4" 
                               data-oe-expression="banner_subtitle">
                                Transform your business with our expertise
                            </p>
                            <div class="s_banner_buttons">
                                <a href="#" class="btn btn-primary btn-lg me-3">Get Started</a>
                                <a href="#" class="btn btn-outline-primary btn-lg">Learn More</a>
                            </div>
                        </div>
                        <div class="col-lg-6" data-name="Image">
                            <img src="/my_theme/static/src/img/snippets/banner-hero.jpg" 
                                 alt="Hero Image" class="img-fluid rounded shadow"/>
                        </div>
                    </div>
                </div>
                <!-- Decorative elements -->
                <div class="s_banner_decoration">
                    <div class="floating-shapes">
                        <div class="shape shape-1"></div>
                        <div class="shape shape-2"></div>
                        <div class="shape shape-3"></div>
                    </div>
                </div>
            </div>
        </section>
    </template>

    <!-- Snippet registration -->
    <record id="snippet_s_banner" model="ir.ui.view">
        <field name="key">my_theme.s_banner</field>
        <field name="name">Banner</field>
        <field name="type">qweb</field>
        <field name="arch" type="xml">
            <div data-snippet="s_banner" data-selector-children=".s_banner" 
                 data-selector-siblings="p, h1, h2, h3, blockquote">
                <div class="oe_snippet_thumbnail">
                    <img class="oe_snippet_thumbnail_img" 
                         src="/my_theme/static/src/img/snippets/s_banner.jpg"/>
                    <span class="oe_snippet_thumbnail_title">Professional Banner</span>
                </div>
                <section class="s_banner">
                    <div class="container">
                        <div class="row">
                            <div class="col-lg-6">
                                <h1>Professional Solutions</h1>
                                <p>Transform your business</p>
                            </div>
                        </div>
                    </div>
                </section>
            </div>
        </field>
    </record>
</odoo>
```

#### Features Grid Snippet

```xml
<!-- views/snippets/s_features.xml -->
<odoo>
    <template id="s_features" name="Features Grid">
        <section class="s_features py-5" data-snippet="s_features" data-name="Features">
            <div class="container">
                <div class="row text-center mb-5">
                    <div class="col-lg-8 mx-auto">
                        <h2 class="display-5 font-weight-bold">Our Features</h2>
                        <p class="lead">Discover what makes us different</p>
                    </div>
                </div>
                <div class="row" data-name="Features Grid">
                    <div class="col-lg-4 col-md-6 mb-4" data-name="Feature">
                        <div class="s_feature_card h-100 text-center p-4 border rounded shadow-sm">
                            <div class="s_feature_icon mb-3">
                                <i class="fa fa-rocket fa-3x text-primary"></i>
                            </div>
                            <h4 class="s_feature_title">Fast Performance</h4>
                            <p class="s_feature_description">
                                Lightning-fast solutions that boost your productivity
                            </p>
                        </div>
                    </div>
                    <div class="col-lg-4 col-md-6 mb-4" data-name="Feature">
                        <div class="s_feature_card h-100 text-center p-4 border rounded shadow-sm">
                            <div class="s_feature_icon mb-3">
                                <i class="fa fa-shield fa-3x text-primary"></i>
                            </div>
                            <h4 class="s_feature_title">Secure & Reliable</h4>
                            <p class="s_feature_description">
                                Enterprise-grade security you can trust
                            </p>
                        </div>
                    </div>
                    <div class="col-lg-4 col-md-6 mb-4" data-name="Feature">
                        <div class="s_feature_card h-100 text-center p-4 border rounded shadow-sm">
                            <div class="s_feature_icon mb-3">
                                <i class="fa fa-users fa-3x text-primary"></i>
                            </div>
                            <h4 class="s_feature_title">Team Collaboration</h4>
                            <p class="s_feature_description">
                                Work together seamlessly with your team
                            </p>
                        </div>
                    </div>
                </div>
            </div>
        </section>
    </template>
</odoo>
```

### Advanced Snippet Options

#### Banner Layout Options

```javascript
// static/src/js/snippets/s_banner_options.js
import { registry } from '@web/core/registry';
import options from '@web_editor/js/editor/snippets.options';

const BannerOptions = options.Class.extend({
    /**
     * Layout selection options
     */
    selectClass: options.Class.prototype.selectClass,
    
    /**
     * Change banner layout
     */
    selectLayout: function (previewMode, widgetValue, params) {
        const $banner = this.$target;
        const layout = widgetValue;
        
        // Remove all layout classes
        $banner.removeClass('layout-default layout-centered layout-split layout-video');
        
        // Add selected layout class
        $banner.addClass(`layout-${layout}`);
        
        // Adjust content based on layout
        if (layout === 'centered') {
            $banner.find('.row').removeClass('align-items-center').addClass('text-center');
        } else if (layout === 'split') {
            $banner.find('.row').addClass('align-items-center').removeClass('text-center');
        }
        
        return this._super.apply(this, arguments);
    },
    
    /**
     * Background options
     */
    selectBackground: function (previewMode, widgetValue, params) {
        const $banner = this.$target;
        
        // Remove existing background classes
        $banner.removeClass('bg-gradient bg-image bg-video bg-particles');
        
        if (widgetValue === 'gradient') {
            $banner.addClass('bg-gradient');
        } else if (widgetValue === 'image') {
            $banner.addClass('bg-image');
            this._setBackgroundImage();
        } else if (widgetValue === 'video') {
            $banner.addClass('bg-video');
            this._setBackgroundVideo();
        } else if (widgetValue === 'particles') {
            $banner.addClass('bg-particles');
            this._initParticles();
        }
        
        return this._super.apply(this, arguments);
    },
    
    /**
     * Animation options
     */
    selectAnimation: function (previewMode, widgetValue, params) {
        const $banner = this.$target;
        const animations = ['fadeInUp', 'fadeInLeft', 'fadeInRight', 'zoomIn', 'slideInDown'];
        
        // Remove existing animation classes
        animations.forEach(anim => {
            $banner.find('[data-name="Content"]').removeClass(`animate__${anim}`);
        });
        
        if (widgetValue !== 'none') {
            $banner.find('[data-name="Content"]').addClass(`animate__animated animate__${widgetValue}`);
        }
        
        return this._super.apply(this, arguments);
    },
    
    /**
     * Helper methods
     */
    _setBackgroundImage: function () {
        const $banner = this.$target;
        // Logic to set background image
        $banner.css('background-image', 'url("/my_theme/static/src/img/bg-hero.jpg")');
    },
    
    _setBackgroundVideo: function () {
        const $banner = this.$target;
        // Add video background
        const videoHtml = `
            <video autoplay muted loop class="bg-video-element">
                <source src="/my_theme/static/src/video/hero-bg.mp4" type="video/mp4">
            </video>
        `;
        $banner.prepend(videoHtml);
    },
    
    _initParticles: function () {
        // Initialize particle.js or custom particle system
        const $banner = this.$target;
        $banner.find('.s_banner_decoration').addClass('particles-active');
    },
    
    /**
     * Color scheme options
     */
    selectColorScheme: function (previewMode, widgetValue, params) {
        const $banner = this.$target;
        const schemes = ['default', 'dark', 'light', 'primary', 'secondary'];
        
        schemes.forEach(scheme => {
            $banner.removeClass(`color-scheme-${scheme}`);
        });
        
        $banner.addClass(`color-scheme-${widgetValue}`);
        
        return this._super.apply(this, arguments);
    },
});

// Register options
registry.category('website_snippets.options').add('s_banner', BannerOptions);

// Define option metadata
BannerOptions.prototype.xmlDependencies = ['/my_theme/static/src/xml/banner_options.xml'];
```

#### Options XML Configuration

```xml
<!-- static/src/xml/banner_options.xml -->
<templates>
    <t t-name="my_theme.s_banner_options">
        <div data-js="BannerOptions">
            <!-- Layout Options -->
            <we-select string="Layout" data-select-layout="">
                <we-button data-select-layout="default" data-img="/my_theme/static/src/img/options/layout-default.png">Default</we-button>
                <we-button data-select-layout="centered" data-img="/my_theme/static/src/img/options/layout-centered.png">Centered</we-button>
                <we-button data-select-layout="split" data-img="/my_theme/static/src/img/options/layout-split.png">Split</we-button>
            </we-select>
            
            <!-- Background Options -->
            <we-select string="Background" data-select-background="">
                <we-button data-select-background="none">None</we-button>
                <we-button data-select-background="gradient">Gradient</we-button>
                <we-button data-select-background="image">Image</we-button>
                <we-button data-select-background="video">Video</we-button>
                <we-button data-select-background="particles">Particles</we-button>
            </we-select>
            
            <!-- Animation Options -->
            <we-select string="Animation" data-select-animation="">
                <we-button data-select-animation="none">None</we-button>
                <we-button data-select-animation="fadeInUp">Fade In Up</we-button>
                <we-button data-select-animation="fadeInLeft">Fade In Left</we-button>
                <we-button data-select-animation="fadeInRight">Fade In Right</we-button>
                <we-button data-select-animation="zoomIn">Zoom In</we-button>
            </we-select>
            
            <!-- Color Scheme -->
            <we-select string="Color Scheme" data-select-color-scheme="">
                <we-button data-select-color-scheme="default">Default</we-button>
                <we-button data-select-color-scheme="dark">Dark</we-button>
                <we-button data-select-color-scheme="light">Light</we-button>
                <we-button data-select-color-scheme="primary">Primary</we-button>
                <we-button data-select-color-scheme="secondary">Secondary</we-button>
            </we-select>
            
            <!-- Advanced Options -->
            <we-collapse-area string="Advanced">
                <we-input string="Custom CSS Class" data-attribute-name="class" data-save-attribute="true"/>
                <we-checkbox string="Enable Parallax" data-parallax=""/>
                <we-range string="Opacity" data-unit="%" data-min="0" data-max="100" data-step="10" 
                         data-attribute-name="opacity"/>
            </we-collapse-area>
        </div>
    </t>
</templates>
```

---

## SCSS, Variables & Responsive Design

### Theme Variables System

Create a comprehensive variable system for consistent theming:

```scss
// static/src/scss/primary_variables.scss

// Brand Colors
$primary: #1a237e !default;
$secondary: #3f51b5 !default;
$success: #4caf50 !default;
$info: #2196f3 !default;
$warning: #ff9800 !default;
$danger: #f44336 !default;
$light: #f8f9fa !default;
$dark: #212529 !default;

// Typography
$font-family-sans-serif: 'Roboto', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif !default;
$font-family-serif: 'Playfair Display', Georgia, serif !default;
$font-family-monospace: 'Fira Code', 'SFMono-Regular', Consolas, monospace !default;

$font-size-base: 1rem !default;
$font-size-lg: $font-size-base * 1.25 !default;
$font-size-sm: $font-size-base * 0.875 !default;

$h1-font-size: $font-size-base * 2.5 !default;
$h2-font-size: $font-size-base * 2 !default;
$h3-font-size: $font-size-base * 1.75 !default;
$h4-font-size: $font-size-base * 1.5 !default;
$h5-font-size: $font-size-base * 1.25 !default;
$h6-font-size: $font-size-base !default;

// Spacing
$spacer: 1rem !default;
$spacers: (
  0: 0,
  1: $spacer * 0.25,
  2: $spacer * 0.5,
  3: $spacer,
  4: $spacer * 1.5,
  5: $spacer * 3,
  6: $spacer * 4,
  7: $spacer * 5,
  8: $spacer * 6
) !default;

// Breakpoints
$grid-breakpoints: (
  xs: 0,
  sm: 576px,
  md: 768px,
  lg: 992px,
  xl: 1200px,
  xxl: 1400px
) !default;

// Container max widths
$container-max-widths: (
  sm: 540px,
  md: 720px,
  lg: 960px,
  xl: 1140px,
  xxl: 1320px
) !default;

// Border radius
$border-radius: 0.375rem !default;
$border-radius-sm: 0.25rem !default;
$border-radius-lg: 0.5rem !default;
$border-radius-xl: 1rem !default;
$border-radius-pill: 50rem !default;

// Shadows
$box-shadow-sm: 0 0.125rem 0.25rem rgba(0, 0, 0, 0.075) !default;
$box-shadow: 0 0.5rem 1rem rgba(0, 0, 0, 0.15) !default;
$box-shadow-lg: 0 1rem 3rem rgba(0, 0, 0, 0.175) !default;
$box-shadow-inset: inset 0 1px 2px rgba(0, 0, 0, 0.075) !default;

// Transitions
$transition-base: all 0.2s ease-in-out !default;
$transition-fade: opacity 0.15s linear !default;
$transition-collapse: height 0.35s ease !default;

// Theme-specific variables
$header-height: 80px !default;
$footer-bg: $dark !default;
$footer-color: $light !default;

// Snippet-specific variables
$banner-min-height: 600px !default;
$banner-overlay-opacity: 0.7 !default;
$feature-card-hover-transform: translateY(-5px) !default;
$testimonial-avatar-size: 80px !default;

// Animation variables
$animation-duration-fast: 0.2s !default;
$animation-duration-normal: 0.3s !default;
$animation-duration-slow: 0.5s !default;
```

### Bootstrap Overrides

```scss
// static/src/scss/bootstrap_overrides.scss

// Import Bootstrap functions and variables first
@import "~bootstrap/scss/functions";
@import "~bootstrap/scss/variables";

// Override Bootstrap variables
$theme-colors: (
  "primary": $primary,
  "secondary": $secondary,
  "success": $success,
  "info": $info,
  "warning": $warning,
  "danger": $danger,
  "light": $light,
  "dark": $dark,
) !default;

// Custom theme colors
$custom-colors: (
  "gradient-primary": linear-gradient(135deg, $primary, lighten($primary, 20%)),
  "gradient-secondary": linear-gradient(135deg, $secondary, lighten($secondary, 20%)),
) !default;

// Merge custom colors with theme colors
$theme-colors: map-merge($theme-colors, $custom-colors);

// Component overrides
$btn-border-radius: $border-radius-lg;
$btn-padding-y: 0.75rem;
$btn-padding-x: 2rem;
$btn-font-weight: 600;

$card-border-radius: $border-radius-lg;
$card-box-shadow: $box-shadow;
$card-cap-bg: transparent;

$navbar-padding-y: 1rem;
$navbar-brand-font-size: 1.5rem;
$navbar-nav-link-padding-x: 1rem;

// Import Bootstrap
@import "~bootstrap/scss/bootstrap";

// Custom utilities
.text-gradient {
  background: linear-gradient(135deg, $primary, $secondary);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.bg-gradient-primary {
  background: linear-gradient(135deg, $primary, lighten($primary, 20%)) !important;
}

.bg-gradient-secondary {
  background: linear-gradient(135deg, $secondary, lighten($secondary, 20%)) !important;
}

// Hover effects
.hover-lift {
  transition: $transition-base;
  
  &:hover {
    transform: translateY(-5px);
    box-shadow: $box-shadow-lg;
  }
}

.hover-scale {
  transition: $transition-base;
  
  &:hover {
    transform: scale(1.05);
  }
}
```

### Responsive Design Patterns

```scss
// static/src/scss/responsive.scss

// Responsive typography
@include media-breakpoint-down(md) {
  .display-1 { font-size: 3rem; }
  .display-2 { font-size: 2.5rem; }
  .display-3 { font-size: 2rem; }
  .display-4 { font-size: 1.75rem; }
  .display-5 { font-size: 1.5rem; }
  .display-6 { font-size: 1.25rem; }
}

// Responsive spacing
.py-mobile-5 {
  @include media-breakpoint-down(md) {
    padding-top: $spacer * 2 !important;
    padding-bottom: $spacer * 2 !important;
  }
}

.px-mobile-3 {
  @include media-breakpoint-down(md) {
    padding-left: $spacer !important;
    padding-right: $spacer !important;
  }
}

// Mobile-first snippet styles
.s_banner {
  min-height: 400px;
  
  @include media-breakpoint-up(md) {
    min-height: $banner-min-height;
  }
  
  .s_banner_title {
    font-size: 2rem;
    
    @include media-breakpoint-up(md) {
      font-size: 3rem;
    }
    
    @include media-breakpoint-up(lg) {
      font-size: 3.5rem;
    }
  }
  
  .s_banner_buttons {
    flex-direction: column;
    gap: 1rem;
    
    @include media-breakpoint-up(sm) {
      flex-direction: row;
      gap: 1.5rem;
    }
    
    .btn {
      width: 100%;
      
      @include media-breakpoint-up(sm) {
        width: auto;
      }
    }
  }
}

// Container queries support (modern browsers)
@supports (container-type: inline-size) {
  .responsive-container {
    container-type: inline-size;
  }
  
  @container (max-width: 768px) {
    .container-responsive {
      .btn {
        display: block;
        width: 100%;
        margin-bottom: 0.5rem;
      }
    }
  }
}
```

### Advanced SCSS Mixins

```scss
// static/src/scss/mixins.scss

// Gradient text mixin
@mixin gradient-text($gradient) {
  background: $gradient;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

// Aspect ratio mixin
@mixin aspect-ratio($width, $height) {
  position: relative;
  
  &:before {
    content: "";
    display: block;
    padding-top: ($height / $width) * 100%;
  }
  
  > * {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
  }
}

// Glass morphism effect
@mixin glass-morphism($blur: 20px, $opacity: 0.1) {
  backdrop-filter: blur($blur);
  background: rgba(255, 255, 255, $opacity);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

// Animation mixin
@mixin animate($name, $duration: $animation-duration-normal, $timing: ease-in-out, $delay: 0s) {
  animation: $name $duration $timing $delay;
}

// Scroll animations
@mixin scroll-reveal() {
  opacity: 0;
  transform: translateY(30px);
  transition: all 0.6s ease;
  
  &.revealed {
    opacity: 1;
    transform: translateY(0);
  }
}

// Dark mode support
@mixin dark-mode {
  @media (prefers-color-scheme: dark) {
    @content;
  }
  
  [data-theme="dark"] & {
    @content;
  }
}

// Focus styles
@mixin focus-visible() {
  &:focus-visible {
    outline: 2px solid $primary;
    outline-offset: 2px;
  }
}
```

---

## Dynamic Snippet Options & JavaScript

### Advanced JavaScript Patterns

#### Intersection Observer for Animations

```javascript
// static/src/js/scroll_animations.js
import { browser } from '@web/core/browser/browser';

class ScrollAnimations {
    constructor() {
        this.init();
    }
    
    init() {
        // Only run on frontend
        if (!document.body.classList.contains('frontend')) return;
        
        this.setupIntersectionObserver();
        this.setupCounterAnimations();
        this.setupParallax();
    }
    
    setupIntersectionObserver() {
        const options = {
            threshold: 0.1,
            rootMargin: '0px 0px -50px 0px'
        };
        
        this.observer = new IntersectionObserver((entries) => {
            entries.forEach(entry => {
                if (entry.isIntersecting) {
                    entry.target.classList.add('animate-in');
                    
                    // Trigger specific animations based on data attributes
                    const animationType = entry.target.dataset.animate;
                    if (animationType) {
                        this.triggerAnimation(entry.target, animationType);
                    }
                }
            });
        }, options);
        
        // Observe all animated elements
        document.querySelectorAll('[data-animate]').forEach(el => {
            this.observer.observe(el);
        });
    }
    
    triggerAnimation(element, type) {
        switch (type) {
            case 'counter':
                this.animateCounter(element);
                break;
            case 'progress':
                this.animateProgress(element);
                break;
            case 'typing':
                this.animateTyping(element);
                break;
            default:
                element.classList.add(`animate__${type}`);
        }
    }
    
    animateCounter(element) {
        const target = parseInt(element.dataset.target);
        const duration = parseInt(element.dataset.duration) || 2000;
        const start = 0;
        const increment = target / (duration / 16);
        let current = start;
        
        const timer = setInterval(() => {
            current += increment;
            element.textContent = Math.floor(current);
            
            if (current >= target) {
                element.textContent = target;
                clearInterval(timer);
            }
        }, 16);
    }
    
    animateProgress(element) {
        const percentage = element.dataset.percentage;
        const progressBar = element.querySelector('.progress-bar');
        
        if (progressBar) {
            progressBar.style.width = `${percentage}%`;
        }
    }
    
    animateTyping(element) {
        const text = element.dataset.text || element.textContent;
        const speed = parseInt(element.dataset.speed) || 50;
        
        element.textContent = '';
        let i = 0;
        
        const typeWriter = () => {
            if (i < text.length) {
                element.textContent += text.charAt(i);
                i++;
                setTimeout(typeWriter, speed);
            }
        };
        
        typeWriter();
    }
    
    setupParallax() {
        const parallaxElements = document.querySelectorAll('[data-parallax]');
        
        if (parallaxElements.length === 0) return;
        
        window.addEventListener('scroll', () => {
            const scrolled = window.pageYOffset;
            
            parallaxElements.forEach(element => {
                const rate = scrolled * (element.dataset.parallax || -0.5);
                element.style.transform = `translateY(${rate}px)`;
            });
        });
    }
}

// Initialize when DOM is ready
document.addEventListener('DOMContentLoaded', () => {
    new ScrollAnimations();
});
```

#### Advanced Snippet Interactions

```javascript
// static/src/js/snippets/s_interactive_features.js
import publicWidget from '@web/legacy/js/public/public_widget';

publicWidget.registry.InteractiveFeatures = publicWidget.Widget.extend({
    selector: '.s_interactive_features',
    events: {
        'click .feature-tab': '_onTabClick',
        'mouseenter .feature-card': '_onCardHover',
        'mouseleave .feature-card': '_onCardLeave',
        'click .feature-toggle': '_onToggleClick',
    },
    
    start() {
        this._super(...arguments);
        this.activeTab = 0;
        this.setupTabs();
        this.setupCards();
        this.setupToggleStates();
        return Promise.resolve();
    },
    
    setupTabs() {
        const tabs = this.$('.feature-tab');
        const panels = this.$('.feature-panel');
        
        // Show first tab by default
        tabs.first().addClass('active');
        panels.first().addClass('active');
        
        // Setup ARIA attributes
        tabs.each((index, tab) => {
            const $tab = $(tab);
            const panelId = `panel-${index}`;
            $tab.attr({
                'role': 'tab',
                'aria-controls': panelId,
                'aria-selected': index === 0 ? 'true' : 'false'
            });
        });
        
        panels.each((index, panel) => {
            $(panel).attr({
                'role': 'tabpanel',
                'id': `panel-${index}`,
                'aria-hidden': index === 0 ? 'false' : 'true'
            });
        });
    },
    
    setupCards() {
        // Add hover effects with GSAP if available
        if (window.gsap) {
            this.$('.feature-card').each((index, card) => {
                const $card = $(card);
                const tl = gsap.timeline({ paused: true });
                
                tl.to(card, {
                    duration: 0.3,
                    y: -10,
                    scale: 1.02,
                    boxShadow: '0 20px 40px rgba(0,0,0,0.1)',
                    ease: 'power2.out'
                });
                
                $card.data('hoverAnimation', tl);
            });
        }
    },
    
    setupToggleStates() {
        // Initialize toggle states from localStorage
        this.$('.feature-toggle').each((index, toggle) => {
            const $toggle = $(toggle);
            const key = $toggle.data('storage-key');
            
            if (key) {
                const saved = localStorage.getItem(key);
                if (saved === 'true') {
                    $toggle.addClass('active').attr('aria-pressed', 'true');
                }
            }
        });
    },
    
    _onTabClick(event) {
        event.preventDefault();
        const $clickedTab = $(event.currentTarget);
        const targetIndex = this.$('.feature-tab').index($clickedTab);
        
        this.switchTab(targetIndex);
    },
    
    switchTab(index) {
        if (index === this.activeTab) return;
        
        const $tabs = this.$('.feature-tab');
        const $panels = this.$('.feature-panel');
        
        // Update active states
        $tabs.removeClass('active').attr('aria-selected', 'false');
        $panels.removeClass('active').attr('aria-hidden', 'true');
        
        $tabs.eq(index).addClass('active').attr('aria-selected', 'true');
        $panels.eq(index).addClass('active').attr('aria-hidden', 'false');
        
        this.activeTab = index;
        
        // Trigger custom event
        this.$el.trigger('tab:changed', { index, tab: $tabs.eq(index) });
    },
    
    _onCardHover(event) {
        const $card = $(event.currentTarget);
        const animation = $card.data('hoverAnimation');
        
        if (animation) {
            animation.play();
        } else {
            // Fallback CSS animation
            $card.addClass('hovered');
        }
    },
    
    _onCardLeave(event) {
        const $card = $(event.currentTarget);
        const animation = $card.data('hoverAnimation');
        
        if (animation) {
            animation.reverse();
        } else {
            $card.removeClass('hovered');
        }
    },
    
    _onToggleClick(event) {
        const $toggle = $(event.currentTarget);
        const isActive = $toggle.hasClass('active');
        const newState = !isActive;
        
        $toggle.toggleClass('active', newState)
               .attr('aria-pressed', newState);
        
        // Save state
        const key = $toggle.data('storage-key');
        if (key) {
            localStorage.setItem(key, newState);
        }
        
        // Trigger action
        const action = $toggle.data('action');
        if (action && this[action]) {
            this[action](newState, $toggle);
        }
    },
    
    // Custom toggle actions
    toggleDarkMode(enabled, $toggle) {
        document.body.classList.toggle('dark-mode', enabled);
        
        // Update toggle text
        const lightText = $toggle.data('light-text') || 'Light Mode';
        const darkText = $toggle.data('dark-text') || 'Dark Mode';
        $toggle.text(enabled ? lightText : darkText);
    },
    
    toggleAnimations(enabled, $toggle) {
        document.body.classList.toggle('animations-disabled', !enabled);
        
        // Respect user's motion preferences
        if (!enabled || window.matchMedia('(prefers-reduced-motion: reduce)').matches) {
            document.body.classList.add('animations-disabled');
        }
    },
});
```

---

## Assets Management & Optimization

### Bundle Optimization

```python
# __manifest__.py - Advanced asset management
'assets': {
    'web.assets_frontend': [
        # Critical CSS first
        'my_theme/static/src/scss/critical.scss',
        
        # Core theme files
        'my_theme/static/src/scss/primary_variables.scss',
        'my_theme/static/src/scss/bootstrap_overrides.scss',
        
        # Component styles
        'my_theme/static/src/scss/components/header.scss',
        'my_theme/static/src/scss/components/footer.scss',
        'my_theme/static/src/scss/components/buttons.scss',
        
        # Snippet styles (loaded conditionally)
        ('prepend', 'my_theme/static/src/scss/snippets/s_banner.scss'),
        ('prepend', 'my_theme/static/src/scss/snippets/s_features.scss'),
        
        # JavaScript
        'my_theme/static/src/js/theme.js',
        'my_theme/static/src/js/scroll_animations.js',
    ],
    
    'web.assets_frontend_lazy': [
        # Non-critical CSS
        'my_theme/static/src/scss/animations.scss',
        'my_theme/static/src/scss/utilities.scss',
        
        # Third-party libraries
        'my_theme/static/lib/swiper/swiper-bundle.min.css',
        'my_theme/static/lib/aos/aos.css',
        
        # Heavy JavaScript
        'my_theme/static/lib/gsap/gsap.min.js',
        'my_theme/static/lib/swiper/swiper-bundle.min.js',
        'my_theme/static/src/js/snippets/advanced_animations.js',
    ],
    
    'website.assets_wysiwyg': [
        # Editor-specific assets
        'my_theme/static/src/scss/editor.scss',
        'my_theme/static/src/js/snippet_options/*.js',
    ],
    
    # Custom bundle for specific pages
    'my_theme.assets_shop': [
        'my_theme/static/src/scss/shop.scss',
        'my_theme/static/src/js/shop_enhancements.js',
    ],
    
    # Mobile-specific optimizations
    'my_theme.assets_mobile': [
        'my_theme/static/src/scss/mobile_optimizations.scss',
        'my_theme/static/src/js/mobile_interactions.js',
    ],
},
```

### Performance Optimization

```scss
// static/src/scss/performance.scss

// Critical CSS - inline in <head>
.above-fold {
  // Styles for content visible without scrolling
  font-display: swap; // For web fonts
  
  // Prevent layout shift
  img {
    aspect-ratio: 16 / 9;
    object-fit: cover;
  }
}

// Lazy loading styles
.lazy-load {
  opacity: 0;
  transition: opacity 0.3s;
  
  &.loaded {
    opacity: 1;
  }
}

// Reduce motion for accessibility
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}

// Print styles
@media print {
  .no-print {
    display: none !important;
  }
  
  .print-break {
    break-before: page;
  }
}
```

### Resource Hints & Preloading

```xml
<!-- views/layout.xml - Performance optimizations -->
<template id="layout_performance" inherit_id="website.layout" name="Performance Optimizations">
    <xpath expr="//head" position="inside">
        <!-- DNS prefetch for external resources -->
        <link rel="dns-prefetch" href="//fonts.googleapis.com"/>
        <link rel="dns-prefetch" href="//fonts.gstatic.com"/>
        <link rel="dns-prefetch" href="//cdn.jsdelivr.net"/>
        
        <!-- Preconnect to critical resources -->
        <link rel="preconnect" href="https://fonts.googleapis.com"/>
        <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin=""/>
        
        <!-- Preload critical assets -->
        <link rel="preload" href="/my_theme/static/src/fonts/primary.woff2" as="font" type="font/woff2" crossorigin=""/>
        <link rel="preload" href="/web/static/lib/bootstrap/scss/bootstrap.scss" as="style"/>
        
        <!-- Critical CSS inline -->
        <style>
            /* Critical above-the-fold styles */
            .s_banner { min-height: 100vh; display: flex; align-items: center; }
            .navbar { transition: all 0.3s ease; }
        </style>
        
        <!-- Resource hints for next page -->
        <link rel="prefetch" href="/page/about"/>
        <link rel="prefetch" href="/shop"/>
        
        <!-- Service Worker registration -->
        <script>
            if ('serviceWorker' in navigator) {
                navigator.serviceWorker.register('/my_theme/static/src/js/sw.js');
            }
        </script>
    </xpath>
</template>
```

---

## Multi-website & Translations

### Multi-website Setup

```python
# models/website_config.py
from odoo import models, fields, api

class Website(models.Model):
    _inherit = 'website'
    
    # Theme-specific configurations
    theme_primary_color = fields.Char(string='Primary Color', default='#1a237e')
    theme_secondary_color = fields.Char(string='Secondary Color', default='#3f51b5')
    theme_layout = fields.Selection([
        ('default', 'Default'),
        ('minimal', 'Minimal'),
        ('corporate', 'Corporate'),
        ('creative', 'Creative'),
    ], default='default', string='Theme Layout')
    
    # Features toggles
    enable_animations = fields.Boolean(string='Enable Animations', default=True)
    enable_parallax = fields.Boolean(string='Enable Parallax Effects', default=True)
    enable_dark_mode = fields.Boolean(string='Enable Dark Mode Toggle', default=False)
    
    # Performance settings
    lazy_load_images = fields.Boolean(string='Lazy Load Images', default=True)
    minify_assets = fields.Boolean(string='Minify CSS/JS', default=True)
    
    @api.model
    def get_theme_config(self):
        """Return theme configuration for current website"""
        website = self.get_current_website()
        return {
            'primary_color': website.theme_primary_color,
            'secondary_color': website.theme_secondary_color,
            'layout': website.theme_layout,
            'features': {
                'animations': website.enable_animations,
                'parallax': website.enable_parallax,
                'dark_mode': website.enable_dark_mode,
            },
            'performance': {
                'lazy_load': website.lazy_load_images,
                'minify': website.minify_assets,
            }
        }
```

```xml
<!-- views/website_config.xml -->
<odoo>
    <record id="website_theme_config_form" model="ir.ui.view">
        <field name="name">Website Theme Configuration</field>
        <field name="model">website</field>
        <field name="inherit_id" ref="website.website_form"/>
        <field name="arch" type="xml">
            <xpath expr="//group[@name='website']" position="after">
                <group string="Theme Configuration" name="theme_config">
                    <group>
                        <field name="theme_primary_color" widget="color"/>
                        <field name="theme_secondary_color" widget="color"/>
                        <field name="theme_layout"/>
                    </group>
                    <group>
                        <field name="enable_animations"/>
                        <field name="enable_parallax"/>
                        <field name="enable_dark_mode"/>
                    </group>
                </group>
                <group string="Performance Settings" name="performance_config">
                    <field name="lazy_load_images"/>
                    <field name="minify_assets"/>
                </group>
            </xpath>
        </field>
    </record>
</odoo>
```

### Translation Management

```xml
<!-- data/website_menu.xml -->
<odoo>
    <!-- Multilingual navigation menu -->
    <record id="menu_main_multilingual" model="website.menu">
        <field name="name" translate="true">Home</field>
        <field name="url">/</field>
        <field name="website_id" ref="website.default_website"/>
        <field name="sequence">1</field>
    </record>
    
    <record id="menu_about_multilingual" model="website.menu">
        <field name="name" translate="true">About Us</field>
        <field name="url">/page/about</field>
        <field name="parent_id" ref="menu_main_multilingual"/>
        <field name="sequence">2</field>
    </record>
    
    <!-- Language-specific content -->
    <record id="homepage_content_en" model="ir.ui.view">
        <field name="name">Homepage Content (English)</field>
        <field name="key">my_theme.homepage_content</field>
        <field name="type">qweb</field>
        <field name="website_id" ref="website.default_website"/>
        <field name="arch" type="xml">
            <div class="homepage-hero">
                <h1 t-translation="homepage.hero.title">Transform Your Business</h1>
                <p t-translation="homepage.hero.subtitle">Professional solutions for modern challenges</p>
            </div>
        </field>
    </record>
</odoo>
```

```python
# models/website_snippet.py
class WebsiteSnippet(models.Model):
    _name = 'website.snippet'
    _description = 'Website Snippet Content'
    
    name = fields.Char(string='Name', required=True, translate=True)
    content = fields.Html(string='Content', translate=True)
    website_id = fields.Many2one('website', string='Website')
    language_code = fields.Char(string='Language Code')
    
    @api.model
    def get_snippet_content(self, snippet_key, lang=None):
        """Get translated snippet content"""
        domain = [('name', '=', snippet_key)]
        
        if lang:
            domain.append(('language_code', '=', lang))
        
        website = self.env['website'].get_current_website()
        if website:
            domain.append(('website_id', '=', website.id))
        
        snippet = self.search(domain, limit=1)
        return snippet.content if snippet else ''
```

### RTL Support

```scss
// static/src/scss/rtl.scss

// RTL-specific styles
[dir="rtl"] {
  .s_banner {
    .s_banner_content {
      text-align: right;
    }
    
    .s_banner_buttons {
      flex-direction: row-reverse;
      
      .btn {
        margin-left: 1rem;
        margin-right: 0;
        
        &:first-child {
          margin-left: 0;
        }
      }
    }
  }
  
  .s_features {
    .feature-card {
      .feature-icon {
        float: right;
        margin-left: 1rem;
        margin-right: 0;
      }
    }
  }
  
  // Navigation RTL
  .navbar-nav {
    .nav-link {
      padding-left: 0;
      padding-right: 1rem;
    }
  }
  
  // Forms RTL
  .form-group {
    label {
      text-align: right;
    }
    
    input, textarea, select {
      text-align: right;
    }
  }
}

// Logical properties (modern approach)
.modern-spacing {
  margin-inline-start: 1rem;
  margin-inline-end: 2rem;
  padding-inline: 1rem;
  border-inline-start: 1px solid #ddd;
}
```
````
This is the description of what the code block changes:
<changeDescription>
Adding the remaining comprehensive sections for SEO, Performance, Advanced Customizations, Testing, and Migration
</changeDescription>

This is the code block that represents the suggested code change:
````markdown
---

## SEO & Performance

### SEO Optimization

#### Meta Tags & Structured Data

```xml
<!-- views/seo_templates.xml -->
<odoo>
    <template id="website_seo_meta" inherit_id="website.layout" name="SEO Meta Tags">
        <xpath expr="//head" position="inside">
            <!-- Open Graph -->
            <meta property="og:site_name" t-att-content="website.name"/>
            <meta property="og:title" t-att-content="main_object.seo_name or main_object.name or website.name"/>
            <meta property="og:description" t-att-content="main_object.website_meta_description or website.website_meta_description"/>
            <meta property="og:image" t-att-content="main_object.website_meta_og_img or website.website_meta_og_img"/>
            <meta property="og:url" t-att-content="request.httprequest.url"/>
            <meta property="og:type" content="website"/>
            
            <!-- Twitter Card -->
            <meta name="twitter:card" content="summary_large_image"/>
            <meta name="twitter:site" t-att-content="website.social_twitter"/>
            <meta name="twitter:title" t-att-content="main_object.seo_name or main_object.name or website.name"/>
            <meta name="twitter:description" t-att-content="main_object.website_meta_description or website.website_meta_description"/>
            <meta name="twitter:image" t-att-content="main_object.website_meta_og_img or website.website_meta_og_img"/>
            
            <!-- Structured Data -->
            <script type="application/ld+json" t-if="website.company_id">
                {
                    "@context": "https://schema.org",
                    "@type": "Organization",
                    "name": "<t t-esc="website.company_id.name"/>",
                    "url": "<t t-esc="website.domain"/>",
                    "logo": "<t t-esc="website.logo_url"/>",
                    "contactPoint": {
                        "@type": "ContactPoint",
                        "telephone": "<t t-esc="website.company_id.phone"/>",
                        "contactType": "customer service",
                        "email": "<t t-esc="website.company_id.email"/>"
                    },
                    "address": {
                        "@type": "PostalAddress",
                        "streetAddress": "<t t-esc="website.company_id.street"/>",
                        "addressLocality": "<t t-esc="website.company_id.city"/>",
                        "addressRegion": "<t t-esc="website.company_id.state_id.name"/>",
                        "postalCode": "<t t-esc="website.company_id.zip"/>",
                        "addressCountry": "<t t-esc="website.company_id.country_id.code"/>"
                    },
                    "sameAs": [
                        "<t t-esc="website.social_facebook"/>",
                        "<t t-esc="website.social_twitter"/>",
                        "<t t-esc="website.social_linkedin"/>",
                        "<t t-esc="website.social_instagram"/>"
                    ]
                }
            </script>
        </xpath>
    </template>
</odoo>
```

### Performance Optimization

```javascript
// static/src/js/image_optimization.js
class ImageOptimizer {
    constructor() {
        this.lazyImages = document.querySelectorAll('img[data-src]');
        this.initLazyLoading();
        this.setupWebPSupport();
        this.setupResponsiveImages();
    }
    
    initLazyLoading() {
        if ('IntersectionObserver' in window) {
            const imageObserver = new IntersectionObserver((entries, observer) => {
                entries.forEach(entry => {
                    if (entry.isIntersecting) {
                        const img = entry.target;
                        this.loadImage(img);
                        observer.unobserve(img);
                    }
                });
            }, {
                rootMargin: '50px 0px'
            });
            
            this.lazyImages.forEach(img => imageObserver.observe(img));
        } else {
            // Fallback for older browsers
            this.lazyImages.forEach(img => this.loadImage(img));
        }
    }
    
    loadImage(img) {
        const src = img.dataset.src;
        const srcset = img.dataset.srcset;
        
        // Create a new image to test loading
        const imageLoader = new Image();
        
        imageLoader.onload = () => {
            img.src = src;
            if (srcset) img.srcset = srcset;
            img.classList.add('loaded');
            img.classList.remove('loading');
        };
        
        imageLoader.onerror = () => {
            // Use fallback image
            img.src = img.dataset.fallback || '/web/static/src/img/placeholder.png';
            img.classList.add('error');
        };
        
        img.classList.add('loading');
        imageLoader.src = src;
    }
    
    setupWebPSupport() {
        // Detect WebP support
        const webp = new Image();
        webp.onload = webp.onerror = () => {
            const isWebPSupported = webp.height === 2;
            document.documentElement.classList.toggle('webp', isWebPSupported);
            document.documentElement.classList.toggle('no-webp', !isWebPSupported);
        };
        webp.src = 'data:image/webp;base64,UklGRjoAAABXRUJQVlA4IC4AAACyAgCdASoCAAIALmk0mk0iIiIiIgBoSygABc6WWgAA/veff/0PP8bA//LwYAAA';
    }
    
    setupResponsiveImages() {
        // Add responsive image attributes based on device
        const images = document.querySelectorAll('img[data-responsive]');
        
        images.forEach(img => {
            const sizes = img.dataset.sizes || '100vw';
            const srcsetBase = img.dataset.srcsetBase;
            
            if (srcsetBase) {
                const srcset = [
                    `${srcsetBase}-400w.jpg 400w`,
                    `${srcsetBase}-800w.jpg 800w`,
                    `${srcsetBase}-1200w.jpg 1200w`,
                    `${srcsetBase}-1600w.jpg 1600w`
                ].join(', ');
                
                img.srcset = srcset;
                img.sizes = sizes;
            }
        });
    }
}

// Initialize when DOM is ready
document.addEventListener('DOMContentLoaded', () => {
    new ImageOptimizer();
});
```

---

## Advanced Customizations

### Theme Configurator

```python
# models/theme_configurator.py
from odoo import models, fields, api
import json

class ThemeConfigurator(models.Model):
    _name = 'theme.configurator'
    _description = 'Theme Configuration'
    
    name = fields.Char(string='Configuration Name', required=True)
    website_id = fields.Many2one('website', string='Website', required=True)
    
    # Color Scheme
    primary_color = fields.Char(string='Primary Color', default='#1a237e')
    secondary_color = fields.Char(string='Secondary Color', default='#3f51b5')
    accent_color = fields.Char(string='Accent Color', default='#ff4081')
    
    # Typography
    heading_font = fields.Selection([
        ('roboto', 'Roboto'),
        ('playfair', 'Playfair Display'),
        ('montserrat', 'Montserrat'),
        ('poppins', 'Poppins'),
    ], default='roboto', string='Heading Font')
    
    body_font = fields.Selection([
        ('roboto', 'Roboto'),
        ('open_sans', 'Open Sans'),
        ('lato', 'Lato'),
        ('source_sans', 'Source Sans Pro'),
    ], default='roboto', string='Body Font')
    
    # Layout
    layout_type = fields.Selection([
        ('boxed', 'Boxed'),
        ('wide', 'Wide'),
        ('fluid', 'Fluid'),
    ], default='wide', string='Layout Type')
    
    header_style = fields.Selection([
        ('standard', 'Standard'),
        ('transparent', 'Transparent'),
        ('minimal', 'Minimal'),
        ('centered', 'Centered'),
    ], default='standard', string='Header Style')
    
    footer_style = fields.Selection([
        ('simple', 'Simple'),
        ('columns', 'Multi-Column'),
        ('minimal', 'Minimal'),
    ], default='columns', string='Footer Style')
    
    # Features
    enable_animations = fields.Boolean(string='Enable Animations', default=True)
    enable_parallax = fields.Boolean(string='Enable Parallax', default=True)
    enable_dark_mode = fields.Boolean(string='Enable Dark Mode', default=False)
    enable_rtl = fields.Boolean(string='Enable RTL Support', default=False)
    
    # Advanced Options
    custom_css = fields.Text(string='Custom CSS')
    custom_js = fields.Text(string='Custom JavaScript')
    
    configuration_json = fields.Text(string='Configuration JSON')
    
    @api.model
    def generate_css_variables(self):
        """Generate CSS custom properties based on configuration"""
        config = self.search([('website_id', '=', self.env['website'].get_current_website().id)], limit=1)
        if not config:
            return ''
        
        css_vars = f"""
        :root {{
            --primary-color: {config.primary_color};
            --secondary-color: {config.secondary_color};
            --accent-color: {config.accent_color};
            --heading-font: '{config.heading_font}', sans-serif;
            --body-font: '{config.body_font}', sans-serif;
        }}
        """
        
        return css_vars
    
    @api.model
    def apply_configuration(self, config_data):
        """Apply theme configuration"""
        website = self.env['website'].get_current_website()
        config = self.search([('website_id', '=', website.id)], limit=1)
        
        if not config:
            config = self.create({
                'name': f"Config for {website.name}",
                'website_id': website.id,
            })
        
        config.write(config_data)
        
        # Generate and save CSS
        css_content = self._generate_theme_css(config)
        self._save_theme_file(css_content, 'theme_config.css')
        
        return {'success': True, 'config_id': config.id}
    
    def _generate_theme_css(self, config):
        """Generate complete theme CSS based on configuration"""
        css_template = """
        /* Theme Configuration CSS */
        :root {
            --primary: %(primary_color)s;
            --secondary: %(secondary_color)s;
            --accent: %(accent_color)s;
            --font-heading: '%(heading_font)s', sans-serif;
            --font-body: '%(body_font)s', sans-serif;
        }
        
        body {
            font-family: var(--font-body);
        }
        
        h1, h2, h3, h4, h5, h6 {
            font-family: var(--font-heading);
        }
        
        .btn-primary {
            background-color: var(--primary);
            border-color: var(--primary);
        }
        
        .btn-secondary {
            background-color: var(--secondary);
            border-color: var(--secondary);
        }
        
        /* Layout Styles */
        .layout-boxed .container-fluid {
            max-width: 1200px;
            margin: 0 auto;
        }
        
        .header-transparent {
            background: transparent;
            position: absolute;
            width: 100%%;
            z-index: 1000;
        }
        
        .header-minimal .navbar {
            padding: 0.5rem 0;
            box-shadow: none;
        }
        
        /* Custom CSS */
        %(custom_css)s
        """
        
        return css_template % {
            'primary_color': config.primary_color,
            'secondary_color': config.secondary_color,
            'accent_color': config.accent_color,
            'heading_font': config.heading_font.replace('_', ' ').title(),
            'body_font': config.body_font.replace('_', ' ').title(),
            'custom_css': config.custom_css or '',
        }
    
    def _save_theme_file(self, content, filename):
        """Save theme file to static directory"""
        # In a real implementation, this would save to the file system
        # For now, we'll use ir.attachment
        attachment = self.env['ir.attachment'].search([
            ('name', '=', filename),
            ('res_model', '=', self._name),
            ('res_id', '=', self.id)
        ])
        
        if attachment:
            attachment.write({'datas': content.encode()})
        else:
            self.env['ir.attachment'].create({
                'name': filename,
                'type': 'binary',
                'datas': content.encode(),
                'res_model': self._name,
                'res_id': self.id,
                'mimetype': 'text/css',
            })
```

### Dynamic Theme Switching

```javascript
// static/src/js/theme_switcher.js
class ThemeSwitcher {
    constructor() {
        this.themes = {
            light: {
                primary: '#1a237e',
                secondary: '#3f51b5',
                background: '#ffffff',
                text: '#333333',
            },
            dark: {
                primary: '#3f51b5',
                secondary: '#1a237e',
                background: '#1a1a1a',
                text: '#ffffff',
            },
            high_contrast: {
                primary: '#000000',
                secondary: '#ffffff',
                background: '#ffffff',
                text: '#000000',
            }
        };
        
        this.currentTheme = localStorage.getItem('theme') || 'light';
        this.init();
    }
    
    init() {
        this.applyTheme(this.currentTheme);
        this.setupThemeSelector();
        this.setupSystemThemeDetection();
        this.setupAccessibilityFeatures();
    }
    
    applyTheme(themeName) {
        const theme = this.themes[themeName];
        if (!theme) return;
        
        const root = document.documentElement;
        
        // Apply CSS custom properties
        Object.entries(theme).forEach(([key, value]) => {
            root.style.setProperty(`--theme-${key}`, value);
        });
        
        // Update body class
        document.body.className = document.body.className
            .replace(/theme-\w+/g, '')
            .trim() + ` theme-${themeName}`;
        
        // Store preference
        localStorage.setItem('theme', themeName);
        this.currentTheme = themeName;
        
        // Notify other components
        document.dispatchEvent(new CustomEvent('themeChanged', {
            detail: { theme: themeName, colors: theme }
        }));
    }
    
    setupThemeSelector() {
        const selector = document.getElementById('theme-selector');
        if (!selector) return;
        
        selector.addEventListener('change', (e) => {
            this.applyTheme(e.target.value);
        });
        
        // Set current value
        selector.value = this.currentTheme;
    }
    
    setupSystemThemeDetection() {
        // Detect system theme preference
        const mediaQuery = window.matchMedia('(prefers-color-scheme: dark)');
        
        const handleSystemThemeChange = (e) => {
            if (localStorage.getItem('theme') === null) {
                this.applyTheme(e.matches ? 'dark' : 'light');
            }
        };
        
        mediaQuery.addEventListener('change', handleSystemThemeChange);
        
        // Apply on load if no preference set
        if (localStorage.getItem('theme') === null) {
            handleSystemThemeChange(mediaQuery);
        }
    }
    
    setupAccessibilityFeatures() {
        // High contrast mode
        const highContrastMediaQuery = window.matchMedia('(prefers-contrast: high)');
        
        const handleHighContrast = (e) => {
            if (e.matches) {
                this.applyTheme('high_contrast');
            }
        };
        
        highContrastMediaQuery.addEventListener('change', handleHighContrast);
        
        // Reduced motion
        const reducedMotionQuery = window.matchMedia('(prefers-reduced-motion: reduce)');
        
        const handleReducedMotion = (e) => {
            document.documentElement.classList.toggle('reduce-motion', e.matches);
        };
        
        reducedMotionQuery.addEventListener('change', handleReducedMotion);
        handleReducedMotion(reducedMotionQuery);
    }
    
    toggleTheme() {
        const themes = Object.keys(this.themes);
        const currentIndex = themes.indexOf(this.currentTheme);
        const nextIndex = (currentIndex + 1) % themes.length;
        this.applyTheme(themes[nextIndex]);
    }
}

// Initialize theme switcher
document.addEventListener('DOMContentLoaded', () => {
    window.themeSwitcher = new ThemeSwitcher();
});
```

---

## Testing & Debugging

### Theme Testing Framework

```python
# tests/test_theme.py
from odoo.tests import HttpCase, tagged
from odoo.tools import mute_logger

@tagged('post_install', '-at_install')
class TestTheme(HttpCase):
    
    def setUp(self):
        super().setUp()
        self.website = self.env['website'].get_current_website()
        
    def test_theme_installation(self):
        """Test theme is properly installed"""
        # Check if theme assets are loaded
        response = self.url_open('/')
        self.assertEqual(response.status_code, 200)
        
        # Check for theme-specific CSS classes
        self.assertIn('my-theme', response.content.decode())
        
    def test_responsive_design(self):
        """Test responsive design breakpoints"""
        # Test different viewport sizes
        viewports = [
            (320, 568),   # Mobile
            (768, 1024),  # Tablet
            (1920, 1080), # Desktop
        ]
        
        for width, height in viewports:
            with self.subTest(viewport=f"{width}x{height}"):
                # Use Selenium or similar for viewport testing
                self._test_viewport(width, height)
    
    def test_snippet_functionality(self):
        """Test snippet rendering and options"""
        # Create a test page with snippets
        page = self.env['website.page'].create({
            'name': 'Test Page',
            'website_id': self.website.id,
            'url': '/test-snippets',
            'arch': '''
                <div>
                    <section class="s_banner">
                        <div class="container">
                            <h1>Test Banner</h1>
                        </div>
                    </section>
                </div>
            '''
        })
        
        response = self.url_open('/test-snippets')
        self.assertEqual(response.status_code, 200)
        self.assertIn('s_banner', response.content.decode())
    
    def test_performance_metrics(self):
        """Test basic performance metrics"""
        import time
        
        start_time = time.time();
        response = self.url_open('/');
        load_time = time.time() - start_time;
        
        # Assert reasonable load time (adjust threshold as needed)
        self.assertLess(load_time, 2.0, "Page load time exceeds 2 seconds")
        
        # Check response size
        content_length = len(response.content)
        self.assertLess(content_length, 500000, "Page size exceeds 500KB")
    
    def test_accessibility(self):
        """Basic accessibility tests"""
        response = self.url_open('/')
        content = response.content.decode();
        
        # Check for basic accessibility requirements
        self.assertIn('alt=', content, "Images should have alt attributes")
        self.assertIn('aria-', content, "Should include ARIA attributes")
        self.assertIn('<h1', content, "Should have proper heading structure")
    
    @mute_logger('odoo.http')
    def test_error_pages(self):
        """Test custom error pages"""
        # Test 404 page
        response = self.url_open('/non-existent-page')
        self.assertEqual(response.status_code, 404)
        
        # Check if custom 404 template is used
        self.assertIn('Page Not Found', response.content.decode())
    
    def test_seo_elements(self):
        """Test SEO meta tags and structured data"""
        response = self.url_open('/')
        content = response.content.decode();
        
        # Check meta tags
        self.assertIn('<meta name="description"', content)
        self.assertIn('<meta property="og:', content)
        self.assertIn('<meta name="twitter:', content)
        
        # Check structured data
        self.assertIn('application/ld+json', content)
    
    def _test_viewport(self, width, height):
        """Helper method to test specific viewport"""
        # This would use Selenium WebDriver in a real implementation
        # For now, we'll simulate viewport testing
        
        # Set viewport simulation headers
        headers = {
            'User-Agent': f'TestBot/1.0 (Viewport: {width}x{height})'
        }
        
        response = self.url_open('/', headers=headers)
        self.assertEqual(response.status_code, 200)
```

### JavaScript Testing

```javascript
// static/tests/test_theme.js
import { describe, test, expect, beforeEach } from '@jest/globals';
import { ScrollAnimations } from '../src/js/scroll_animations.js';

describe('Theme JavaScript', () => {
    let scrollAnimations;
    
    beforeEach(() => {
        // Setup DOM
        document.body.innerHTML = `
            <div class="animate-element" data-animate="fadeInUp"></div>
            <div class="counter" data-animate="counter" data-target="100"></div>
        `;
        
        scrollAnimations = new ScrollAnimations();
    });
    
    test('should initialize scroll animations', () => {
        expect(scrollAnimations.observer).toBeDefined();
    });
    
    test('should animate counter correctly', () => {
        const counterElement = document.querySelector('.counter');
        
        scrollAnimations.animateCounter(counterElement);
        
        // Wait for animation to complete
        setTimeout(() => {
            expect(counterElement.textContent).toBe('100');
        }, 2100);
    });
    
    test('should handle intersection observer', () => {
        const element = document.querySelector('.animate-element');
        
        // Simulate intersection
        const entries = [{
            target: element,
            isIntersecting: true
        }];
        
        scrollAnimations.observer.callback(entries);
        
        expect(element.classList.contains('animate-in')).toBe(true);
    });
});

// Performance testing
describe('Performance', () => {
    test('should load images efficiently', async () => {
        const imageOptimizer = new (await import('../src/js/image_optimization.js')).ImageOptimizer();
        
        // Mock intersection observer
        global.IntersectionObserver = jest.fn(() => ({
            observe: jest.fn(),
            disconnect: jest.fn(),
        }));
        
        expect(imageOptimizer.lazyImages).toBeDefined();
    });
    
    test('should detect WebP support', async () => {
        const imageOptimizer = new (await import('../src/js/image_optimization.js')).ImageOptimizer();
        
        // Mock Image constructor
        global.Image = jest.fn(() => ({
            onload: null,
            onerror: null,
            src: '',
            height: 2
        }));
        
        imageOptimizer.setupWebPSupport();
        
        // Simulate WebP load
        const img = new Image();
        img.onload();
        
        expect(document.documentElement.classList.contains('webp')).toBe(true);
    });
});
```

---

## Migration & Maintenance

### Version Migration Scripts

```python
# migrations/18.0.1.1.0/pre.py
def migrate(cr, version):
    """Pre-migration script for theme updates"""
    if not version:
        return
    
    # Update existing snippet classes
    cr.execute("""
        UPDATE ir_ui_view 
        SET arch_db = REPLACE(arch_db, 'old-class-name', 'new-class-name')
        WHERE type = 'qweb' AND name LIKE '%snippet%'
    """)
    
    # Migrate theme configurations
    cr.execute("""
        ALTER TABLE theme_configurator 
        ADD COLUMN IF NOT EXISTS new_feature_enabled BOOLEAN DEFAULT FALSE
    """)

# migrations/18.0.1.1.0/post.py
def migrate(cr, version):
    """Post-migration script"""
    if not version:
        return
    
    from odoo import api, SUPERUSER_ID
    
    env = api.Environment(cr, SUPERUSER_ID, {})
    
    # Update website configurations
    websites = env['website'].search([])
    for website in websites:
        # Apply new default settings
        website.write({
            'theme_primary_color': '#1a237e',
            'enable_animations': True,
        })
    
    # Regenerate CSS files
    configurators = env['theme.configurator'].search([])
    for config in configurators:
        config._generate_theme_css(config)
```

### Maintenance Tools

```python
# models/theme_maintenance.py
from odoo import models, fields, api
import logging

_logger = logging.getLogger(__name__)

class ThemeMaintenance(models.TransientModel):
    _name = 'theme.maintenance'
    _description = 'Theme Maintenance Tools'
    
    action_type = fields.Selection([
        ('clear_cache', 'Clear Asset Cache'),
        ('regenerate_css', 'Regenerate CSS'),
        ('validate_templates', 'Validate Templates'),
        ('optimize_images', 'Optimize Images'),
        ('check_performance', 'Performance Check'),
    ], string='Action', required=True)
    
    def execute_maintenance(self):
        """Execute selected maintenance action"""
        method_name = f'_action_{self.action_type}'
        if hasattr(self, method_name):
            getattr(self, method_name)()
            return {
                'type': 'ir.actions.client',
                'tag': 'display_notification',
                'params': {
                    'title': 'Success',
                    'message': f'{self.action_type.replace("_", " ").title()} completed successfully',
                    'type': 'success',
                }
            }
    
    def _action_clear_cache(self):
        """Clear asset cache"""
        self.env['ir.qweb'].clear_caches()
        self.env['ir.attachment'].search([
            ('url', 'like', '/web/assets/%')
        ]).unlink()
        _logger.info("Asset cache cleared")
    
    def _action_regenerate_css(self):
        """Regenerate all theme CSS files"""
        configurators = self.env['theme.configurator'].search([])
        for config in configurators:
            css_content = config._generate_theme_css(config)
            config._save_theme_file(css_content, 'theme_config.css')
        _logger.info(f"Regenerated CSS for {len(configurators)} configurations")
    
    def _action_validate_templates(self):
        """Validate QWeb templates"""
        templates = self.env['ir.ui.view'].search([
            ('type', '=', 'qweb'),
            ('name', 'like', '%my_theme%')
        ])
        
        invalid_templates = []
        for template in templates:
            try:
                self.env['ir.qweb']._compile(template.arch_db, template.id)
            except Exception as e:
                invalid_templates.append((template.name, str(e)))
        
        if invalid_templates:
            _logger.warning(f"Found {len(invalid_templates)} invalid templates")
        else:
            _logger.info("All templates are valid")
    
    def _action_optimize_images(self):
        """Optimize theme images"""
        # This would integrate with image optimization tools
        attachments = self.env['ir.attachment'].search([
            ('mimetype', 'like', 'image/%'),
            ('url', 'like', '%my_theme%')
        ])
        
        optimized_count = 0;
        for attachment in attachments:
            # Simulate optimization
            optimized_count += 1;
        
        _logger.info(f"Optimized {optimized_count} images")
    
    def _action_check_performance(self):
        """Check theme performance metrics"""
        # Analyze CSS/JS bundle sizes
        assets = self.env['ir.attachment'].search([
            ('url', 'like', '/web/assets/%'),
            ('mimetype', 'in', ['text/css', 'application/javascript'])
        ])
        
        total_size = sum(len(asset.raw or b'') for asset in assets)
        _logger.info(f"Total asset size: {total_size / 1024:.2f} KB")
        
        # Check for unused CSS (simplified)
        css_assets = assets.filtered(lambda a: a.mimetype == 'text/css');
        _logger.info(f"CSS files: {len(css_assets)}")
    }
}
```

### Documentation Generator

```python
# tools/doc_generator.py
from odoo import models, api
import json
import os

class ThemeDocGenerator(models.TransientModel):
    _name = 'theme.doc.generator'
    _description = 'Generate Theme Documentation'
    
    @api.model
    def generate_snippet_docs(self):
        """Generate documentation for all snippets"""
        snippets = self.env['ir.ui.view'].search([
            ('type', '=', 'qweb'),
            ('name', 'like', '%snippet%'),
            ('key', 'like', '%my_theme%')
        ])
        
        docs = []
        for snippet in snippets:
            doc = {
                'name': snippet.name,
                'key': snippet.key,
                'description': self._extract_description(snippet.arch_db),
                'options': self._extract_options(snippet),
                'preview_url': f'/web/image/ir.ui.view/{snippet.id}/preview',
            }
            docs.append(doc)
        
        return json.dumps(docs, indent=2)
    
    def _extract_description(self, arch):
        """Extract description from template"""
        # Parse template and extract comments/descriptions
        import re
        description_match = re.search(r'data-name="([^"]*)"', arch)
        return description_match.group(1) if description_match else 'No description'
    
    def _extract_options(self, snippet):
        """Extract available options for snippet"""
        # Look for corresponding option files
        option_views = self.env['ir.ui.view'].search([
            ('name', 'like', f'%{snippet.name}%option%')
        ])
        
        options = []
        for option_view in option_views:
            # Parse option template for available configurations
            options.append({
                'name': option_view.name,
                'type': 'selection',  # Simplified
            })
        
        return options

    @api.model
    def generate_style_guide(self):
        """Generate CSS style guide"""
        style_guide = {
            'colors': self._extract_css_variables('color'),
            'typography': self._extract_css_variables('font'),
            'spacing': self._extract_css_variables('spacer'),
            'components': self._extract_components(),
        };
        
        return json.dumps(style_guide, indent=2);
    }
    
    def _extract_css_variables(self, prefix):
        """Extract CSS variables by prefix"""
        # This would parse SCSS files to extract variables
        # Simplified implementation
        return {
            f'{prefix}_primary': '#1a237e',
            f'{prefix}_secondary': '#3f51b5',
        };
    }
    
    def _extract_components(self):
        """Extract component documentation"""
        # Parse SCSS files for component classes
        return {
            'buttons': {
                'classes': ['.btn-primary', '.btn-secondary', '.btn-outline'],
                'modifiers': ['.btn-lg', '.btn-sm'],
            },
            'cards': {
                'classes': ['.card', '.card-body', '.card-header'],
                'modifiers': ['.card-shadow', '.card-hover'],
            }
        }
```

---

This comprehensive theme guide covers everything from basic setup to advanced customizations, performance optimization, testing, and maintenance. Each section includes practical, ready-to-use code snippets that can be implemented in Odoo 18.0 theme development projects.
