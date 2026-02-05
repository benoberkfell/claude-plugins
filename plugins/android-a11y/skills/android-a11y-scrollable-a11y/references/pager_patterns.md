# Pager Accessibility Patterns

Reference for HorizontalPager, VerticalPager, ViewPager2, and related components.

## Compose HorizontalPager

### Basic Accessible Pager

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun AccessibleHorizontalPager(
    pages: List<PageData>
) {
    val pagerState = rememberPagerState { pages.size }

    Column {
        HorizontalPager(
            state = pagerState,
            modifier = Modifier.semantics {
                // Announce as pager
                contentDescription = "Image gallery with ${pages.size} pages"
            }
        ) { page ->
            PageContent(
                data = pages[page],
                pageNumber = page + 1,
                totalPages = pages.size
            )
        }

        // Accessible page indicator
        PageIndicator(
            currentPage = pagerState.currentPage,
            totalPages = pages.size
        )
    }
}

@Composable
fun PageContent(
    data: PageData,
    pageNumber: Int,
    totalPages: Int
) {
    Box(
        modifier = Modifier
            .fillMaxSize()
            .semantics(mergeDescendants = true) {
                contentDescription = buildString {
                    append(data.title)
                    append(". ")
                    append("Page $pageNumber of $totalPages")
                }
            }
    ) {
        // Page content
    }
}

@Composable
fun PageIndicator(
    currentPage: Int,
    totalPages: Int
) {
    Text(
        text = "Page ${currentPage + 1} of $totalPages",
        modifier = Modifier.semantics {
            liveRegion = LiveRegionMode.Polite
        }
    )
}
```

### Focus Restoration Pattern

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun PagerWithFocusRestoration(
    pages: List<PageData>
) {
    val pagerState = rememberPagerState { pages.size }
    val focusRequesters = remember {
        List(pages.size) { FocusRequester() }
    }

    // Restore focus when page changes
    LaunchedEffect(pagerState.currentPage) {
        // Wait for composition
        snapshotFlow { pagerState.isScrollInProgress }
            .filter { !it }  // Wait for scroll to settle
            .first()

        // Request focus on new page
        try {
            focusRequesters[pagerState.currentPage].requestFocus()
        } catch (e: IllegalStateException) {
            // FocusRequester not attached yet
        }
    }

    HorizontalPager(state = pagerState) { page ->
        Column(
            modifier = Modifier
                .focusRequester(focusRequesters[page])
                .focusable()
        ) {
            PageContent(pages[page])
        }
    }
}
```

### Pager with Tab Row

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun TabbedPager(
    tabs: List<TabData>
) {
    val pagerState = rememberPagerState { tabs.size }
    val coroutineScope = rememberCoroutineScope()

    Column {
        // Accessible tab row
        TabRow(
            selectedTabIndex = pagerState.currentPage,
            modifier = Modifier.semantics {
                // Tab row semantics handled by TabRow
            }
        ) {
            tabs.forEachIndexed { index, tab ->
                Tab(
                    selected = pagerState.currentPage == index,
                    onClick = {
                        coroutineScope.launch {
                            pagerState.animateScrollToPage(index)
                        }
                    },
                    modifier = Modifier.semantics {
                        // Role.Tab is automatic
                        stateDescription = if (pagerState.currentPage == index)
                            "selected" else "not selected"
                    }
                ) {
                    Text(
                        text = tab.title,
                        modifier = Modifier.padding(16.dp)
                    )
                }
            }
        }

        // Pager content
        HorizontalPager(
            state = pagerState,
            modifier = Modifier.semantics {
                isTraversalGroup = true
            }
        ) { page ->
            tabs[page].content()
        }
    }
}
```

## Compose VerticalPager

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun AccessibleVerticalPager(
    items: List<StoryItem>
) {
    val pagerState = rememberPagerState { items.size }

    VerticalPager(
        state = pagerState,
        modifier = Modifier.semantics {
            contentDescription = "Stories feed. Swipe up or down to navigate."
        }
    ) { page ->
        StoryCard(
            item = items[page],
            modifier = Modifier.semantics {
                contentDescription = buildString {
                    append(items[page].title)
                    append(" by ")
                    append(items[page].author)
                    append(". Story ${page + 1} of ${items.size}")
                }
            }
        )
    }
}
```

## ViewPager2 (Views)

### Basic Setup

```kotlin
class AccessibleViewPager2Activity : AppCompatActivity() {

    private lateinit var viewPager: ViewPager2

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_pager)

        viewPager = findViewById(R.id.viewPager)
        viewPager.adapter = PagerAdapter(pages)

        // Accessibility setup
        setupAccessibility()
    }

    private fun setupAccessibility() {
        viewPager.apply {
            // Ensure accessibility is enabled
            importantForAccessibility = View.IMPORTANT_FOR_ACCESSIBILITY_YES

            // Announce page changes
            registerOnPageChangeCallback(object : ViewPager2.OnPageChangeCallback() {
                override fun onPageSelected(position: Int) {
                    announceForAccessibility(
                        "Page ${position + 1} of ${adapter?.itemCount ?: 0}"
                    )
                }
            })

            // Set content description
            contentDescription = "Image gallery"
        }
    }
}
```

### ViewPager2 Adapter

```kotlin
class AccessiblePagerAdapter(
    private val pages: List<PageData>
) : RecyclerView.Adapter<AccessiblePagerAdapter.PageViewHolder>() {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): PageViewHolder {
        val view = LayoutInflater.from(parent.context)
            .inflate(R.layout.page_item, parent, false)
        return PageViewHolder(view)
    }

    override fun onBindViewHolder(holder: PageViewHolder, position: Int) {
        holder.bind(pages[position], position, pages.size)
    }

    override fun getItemCount() = pages.size

    class PageViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        private val imageView: ImageView = itemView.findViewById(R.id.image)
        private val titleText: TextView = itemView.findViewById(R.id.title)

        fun bind(page: PageData, position: Int, total: Int) {
            imageView.setImageResource(page.imageRes)
            titleText.text = page.title

            // Accessibility
            itemView.contentDescription = buildString {
                append(page.title)
                append(". ")
                append(page.description)
                append(". ")
                append("Page ${position + 1} of $total")
            }

            // Ensure page is focusable
            itemView.isFocusable = true
        }
    }
}
```

### TabLayout + ViewPager2

```kotlin
fun setupTabLayoutWithViewPager2(
    tabLayout: TabLayout,
    viewPager: ViewPager2,
    tabs: List<TabInfo>
) {
    // Attach TabLayout to ViewPager2
    TabLayoutMediator(tabLayout, viewPager) { tab, position ->
        tab.text = tabs[position].title

        // Accessibility
        tab.contentDescription = buildString {
            append(tabs[position].title)
            append(" tab")
            append(", ${position + 1} of ${tabs.size}")
        }
    }.attach()

    // Additional accessibility for tab selection
    tabLayout.addOnTabSelectedListener(object : TabLayout.OnTabSelectedListener {
        override fun onTabSelected(tab: TabLayout.Tab) {
            tabLayout.announceForAccessibility(
                "${tab.text} tab selected"
            )
        }

        override fun onTabUnselected(tab: TabLayout.Tab?) {}
        override fun onTabReselected(tab: TabLayout.Tab?) {}
    })
}
```

## Carousel Patterns

### Compose Carousel

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun AccessibleCarousel(
    items: List<CarouselItem>,
    modifier: Modifier = Modifier
) {
    val pagerState = rememberPagerState { items.size }

    Column(modifier = modifier) {
        // Carousel header
        Text(
            text = "Featured",
            style = MaterialTheme.typography.titleMedium,
            modifier = Modifier.semantics { heading() }
        )

        // Carousel content
        HorizontalPager(
            state = pagerState,
            contentPadding = PaddingValues(horizontal = 32.dp),
            pageSpacing = 16.dp,
            modifier = Modifier.semantics {
                contentDescription = "Featured items carousel"
                isTraversalGroup = true
            }
        ) { page ->
            CarouselCard(
                item = items[page],
                isSelected = page == pagerState.currentPage,
                modifier = Modifier.semantics {
                    contentDescription = buildString {
                        append(items[page].title)
                        if (page == pagerState.currentPage) {
                            append(", currently selected")
                        }
                        append(". ${page + 1} of ${items.size}")
                    }
                }
            )
        }

        // Dot indicator
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp),
            horizontalArrangement = Arrangement.Center
        ) {
            repeat(items.size) { index ->
                Box(
                    modifier = Modifier
                        .size(8.dp)
                        .clip(CircleShape)
                        .background(
                            if (index == pagerState.currentPage)
                                MaterialTheme.colorScheme.primary
                            else
                                MaterialTheme.colorScheme.surfaceVariant
                        )
                        .semantics {
                            // Individual dots not focusable
                            invisibleToUser()
                        }
                )
                if (index < items.size - 1) {
                    Spacer(Modifier.width(8.dp))
                }
            }
        }

        // Accessible position text (for screen readers)
        Text(
            text = "${pagerState.currentPage + 1} of ${items.size}",
            modifier = Modifier
                .semantics { liveRegion = LiveRegionMode.Polite }
                // Visually hidden but available to screen readers
                .alpha(0f)
                .height(0.dp)
        )
    }
}
```

## Common Issues and Fixes

### Issue: Page Content Not Announced

```kotlin
// Problem: No semantics on page
HorizontalPager(state = pagerState) { page ->
    Image(painter = ..., contentDescription = null)  // Silent!
}

// Solution: Add content description
HorizontalPager(state = pagerState) { page ->
    Image(
        painter = ...,
        contentDescription = "Photo ${page + 1}: ${pages[page].description}"
    )
}
```

### Issue: Can't Navigate Between Pages

```kotlin
// Problem: Pager nested in non-scrollable container
Box(Modifier.fillMaxSize()) {
    HorizontalPager(state = pagerState) { ... }
    // Other content blocking gestures
}

// Solution: Proper layering and gesture handling
Box(Modifier.fillMaxSize()) {
    HorizontalPager(
        state = pagerState,
        modifier = Modifier.fillMaxSize()  // Takes full space
    ) { ... }

    // Overlay content doesn't block pager gestures
    Box(
        Modifier
            .align(Alignment.BottomCenter)
            .semantics { isTraversalGroup = true }
    ) {
        // Overlay content
    }
}
```

### Issue: Focus Lost on Page Change

See "Focus Restoration Pattern" above. Key points:
1. Use `FocusRequester` per page
2. Wait for scroll to settle before requesting focus
3. Handle `IllegalStateException` for not-yet-attached requesters

### Issue: Tab + Pager Sync Issues

```kotlin
// Ensure bidirectional sync
LaunchedEffect(pagerState.currentPage) {
    // Pager drives tabs
}

// Tab click drives pager
Tab(
    onClick = {
        coroutineScope.launch {
            pagerState.animateScrollToPage(index)
        }
    }
)
```
