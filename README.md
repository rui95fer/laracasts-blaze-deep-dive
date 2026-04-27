# Blaze Deep-Dive — Laracasts Course Notes

---

## Episode 01 — Why Blaze Exists

- **Blade components have hidden overhead at scale — each one adds up.**
  ```blade
  {{-- A single page using small components everywhere --}}
  <x-button>Add to Cart</x-button>
  <x-icon name="chevron-right" />
  <x-badge :status="$user->role" />
  {{-- Innocent alone. 50 rows × 3 components = 150 renders per page load. --}}
  ```

- **The database is not always the bottleneck — Blade rendering often is.**
  ```bash
  # Before assuming slow queries, profile your view layer
  # Telescope, Debugbar, or even a simple benchmark can reveal Blade as the culprit
  php artisan telescope:install
  ```

- **Blaze is the fix — a drop-in package with performance strategies baked in.**
  ```bash
  # Install Blaze (covered in upcoming episodes)
  composer require livewire/blaze
  ```

---

## Episode 02 — Installing Blaze

- **Blade component loops inside other loops are the #1 performance killer.**
  ```blade
  {{-- A country select inside a loop of 250 users --}}
  @foreach ($users as $user)
      <x-select>
          @foreach ($countries as $country)   {{-- ~200 iterations --}}
              <x-option :value="$country" />  {{-- Blade component --}}
          @endforeach
      </x-select>
  @endforeach
  {{-- 250 users × 200 options = 50,000 Blade component renders --}}
  ```

- **A Blade component that does too much work (query, session, regex) becomes a time bomb when placed inside a loop.**
  ```blade
  {{-- Looks innocent, but fires a DB query 250 times --}}
  @foreach ($users as $user)
      <x-users-online :user="$user" />  {{-- queries last_login on every render --}}
  @endforeach
  ```

- **Install Blaze with a single Composer command — that's it.**
  ```bash
  composer require livewire/blaze
  ```

- **Always clear the compiled view cache after installing or changing anything Blade-related.**
  ```bash
  php artisan view:clear
  # Add this to your deploy script — it belongs there anyway
  ```

- **For Flux users: installing Blaze alone gives a ~25× speed improvement with zero other changes.**
  ```
  Before Blaze (Flux page, 250 users):  ~723ms
  After Blaze  (same page, no changes):  ~35ms
  ```

- **Blaze does not auto-optimize your own custom Blade components — you have to opt them in explicitly.**
  ```blade
  {{-- x-heading, x-text, x-badge etc. are NOT touched by Blaze out of the box --}}
  {{-- Next episode: how to tell Blaze to optimize your own components --}}
  ```

---

## Episode 03 — Three Levels of Optimization

- **Add `@blaze` to the top of a single component to opt it into Blaze's optimized compiler.**
  ```blade
  {{-- resources/views/components/option.blade.php --}}
  @blaze
  <option {{ $attributes }}>{{ $slot }}</option>
  ```
  ```bash
  php artisan view:clear
  ```

- **To optimize all components at once, use `Blaze::optimizeIn()` in your service provider instead of adding `@blaze` everywhere.**
  ```php
  // app/Providers/AppServiceProvider.php
  use Livewire\Blaze\Blaze;

  public function boot(): void
  {
      Blaze::optimizeIn(resource_path('views/components'));
  }
  ```

- **Blaze has three levels of performance — each one goes deeper.**
  ```
  Level 1 — Optimized compiler (default, just add @blaze)
      → ~2× speed improvement, removes most Blade overhead
      → 700ms → 314ms

  Level 2 — Memoization (memo: true)
      → Component renders once, result is reused everywhere else
      → Best for components with expensive work (queries, session lookups)
      → 314ms → 97ms

  Level 3 — Code folding (fold: true)
      → Goes even deeper (covered in later episodes)
      → 97ms → 52ms
  ```

- **Use `memo: true` on any component that does expensive work inside a loop — it renders once and caches the result.**
  ```blade
  {{-- resources/views/components/online-count.blade.php --}}
  @blaze(memo: true)
  {{-- This DB query now only runs once no matter how many times the component is used --}}
  <span>{{ DB::table('logins')->where(...)->count() }} online</span>
  ```

- **Use `fold: true` on static, repeated components like select options to collapse them at compile time.**
  ```blade
  {{-- resources/views/components/option.blade.php --}}
  @blaze(fold: true)
  <option {{ $attributes }}>{{ $slot }}</option>
  ```

- **Summary: Flux users get all three levels for free. Custom components start at Level 1 and you opt into deeper levels as needed.**
  ```
  Flux + Blaze installed:         ~35ms  (all levels automatic)
  Custom + @blaze on everything:  ~314ms  (Level 1 only)
  Custom + memo + fold:            ~52ms  (all three levels)
  ```
