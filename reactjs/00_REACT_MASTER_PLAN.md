# React.js Interview Questions - Master Plan
## 50 Questions | 50 Files | Banking Application Examples

---

## 📋 Project Overview

**Total Questions**: 50  
**Total Files**: 50 (One question per file)  
**Focus**: Banking/E-commerce applications with real-world scenarios  
**Quality Standard**: Production-ready code with architecture-level explanations (~1,500-2,000 lines per file)

**Content Approach**:
- ✅ Detailed explanations with React internals and architecture
- ✅ Banking/E-commerce specific examples (dashboards, payments, micro frontends)
- ✅ Production-ready code with TypeScript and error handling
- ✅ Testing instructions with actual commands
- ✅ DO's and DON'Ts for best practices
- ✅ Performance comparisons and optimization techniques
- ✅ **Last 10 questions focused on Micro Frontend Architecture**

---

## 🎯 Questions Breakdown

### 📘 **BEGINNER LEVEL** (Questions 1-10)

#### **File Q01**: JSX & Virtual DOM
- What is JSX and how does it work?
- Virtual DOM reconciliation algorithm
- Babel transformation process
- React Fiber architecture basics
- Banking Example: Building account dashboard with JSX

#### **File Q02**: Components & Props
- Functional vs Class components
- Props flow and validation
- PropTypes and TypeScript interfaces
- Component composition patterns
- Banking Example: Reusable transaction card component

#### **File Q03**: State Management - useState
- State fundamentals and immutability
- useState hook internals
- State updates and batching
- Multiple state variables vs object state
- Banking Example: Account balance tracker

#### **File Q04**: Lifecycle Methods
- Class component lifecycle phases
- componentDidMount, componentDidUpdate, componentWillUnmount
- Side effects and cleanup
- Migration to hooks
- Banking Example: Real-time balance updates

#### **File Q05**: React Hooks - Basics
- Rules of Hooks and why they exist
- Hook call order and React internals
- useState, useEffect fundamentals
- Hook dependencies
- Banking Example: Transaction form with validation

#### **File Q06**: useEffect Hook
- Effect execution timing
- Dependency array deep dive
- Cleanup functions
- Common pitfalls and solutions
- Banking Example: Fetching transaction history

#### **File Q07**: Event Handling
- Synthetic events system
- Event pooling (legacy) and modern approach
- Event delegation in React
- Preventing default and stopPropagation
- Banking Example: Form submission with validation

#### **File Q08**: Conditional Rendering
- if/else, ternary, && operator
- Switch statements and early returns
- Performance implications
- Avoiding unnecessary renders
- Banking Example: Showing different UI based on account type

#### **File Q09**: Lists & Keys
- Rendering lists with map()
- Why keys are important for reconciliation
- Index as key anti-pattern
- Key selection strategies
- Banking Example: Transaction list with filtering

#### **File Q10**: Forms & Controlled Components
- Controlled vs Uncontrolled components
- Form state management
- Validation patterns
- File uploads and complex inputs
- Banking Example: Multi-step account opening form

---

### 📙 **INTERMEDIATE LEVEL** (Questions 11-20)

#### **File Q11**: Context API
- Context creation and usage
- Provider pattern
- Context vs Props drilling
- Multiple contexts
- Banking Example: Global user authentication context

#### **File Q12**: useReducer Hook
- Reducer pattern in React
- When to use useReducer vs useState
- Complex state logic
- Dispatch and action types
- Banking Example: Shopping cart state management

#### **File Q13**: Custom Hooks
- Creating reusable hooks
- Hook composition
- Testing custom hooks
- Hook best practices
- Banking Example: useTransactions, useAccountBalance hooks

#### **File Q14**: React.memo & useMemo
- Component memoization
- When to use React.memo
- useMemo for expensive calculations
- Referential equality
- Banking Example: Optimizing transaction calculations

#### **File Q15**: useCallback Hook
- Function memoization
- Preventing unnecessary re-renders
- useCallback vs useMemo
- Dependency management
- Banking Example: Optimized event handlers

#### **File Q16**: Refs & useRef
- DOM access with refs
- useRef for mutable values
- forwardRef pattern
- Ref callback pattern
- Banking Example: Focus management in forms

#### **File Q17**: Error Boundaries
- Error boundary implementation
- componentDidCatch and getDerivedStateFromError
- Error recovery strategies
- Logging errors
- Banking Example: Graceful error handling in payment flow

#### **File Q18**: Portals
- ReactDOM.createPortal usage
- Modal dialogs
- Tooltips and dropdowns
- Event bubbling through portals
- Banking Example: Transaction confirmation modal

#### **File Q19**: Higher-Order Components (HOC)
- HOC pattern explained
- Creating and composing HOCs
- Props manipulation
- HOC vs Hooks
- Banking Example: withAuth, withLogging HOCs

#### **File Q20**: Render Props Pattern
- Render props concept
- Function as children
- Component reusability
- Render props vs Hooks
- Banking Example: Reusable data fetching component

---

### 📕 **INTERMEDIATE-ADVANCED LEVEL** (Questions 21-30)

#### **File Q21**: Redux Fundamentals
- Redux architecture (Store, Actions, Reducers)
- Unidirectional data flow
- Pure functions and immutability
- Redux DevTools
- Banking Example: Global banking app state

#### **File Q22**: Redux Toolkit
- configureStore and createSlice
- createAsyncThunk for async logic
- RTK Query basics
- Immer integration
- Banking Example: Modern Redux implementation

#### **File Q23**: Redux Middleware
- Middleware concept and execution flow
- redux-thunk for async actions
- redux-saga for complex side effects
- Custom middleware
- Banking Example: Transaction logging middleware

#### **File Q24**: React Redux Hooks
- useSelector and reselect
- useDispatch for actions
- Typed hooks with TypeScript
- Performance optimization
- Banking Example: Connected banking components

#### **File Q25**: React Router - Basics
- BrowserRouter vs HashRouter
- Route configuration
- Link and NavLink
- Programmatic navigation
- Banking Example: Multi-page banking app

#### **File Q26**: React Router - Advanced
- Protected routes and authentication
- Dynamic routing
- Route guards
- Lazy loading routes
- Banking Example: Secure banking dashboard routing

#### **File Q27**: Form Libraries - Formik
- Formik setup and usage
- Field-level validation
- Form submission handling
- Complex forms
- Banking Example: Account application form

#### **File Q28**: Form Validation - Yup/Zod
- Schema validation
- Custom validators
- Async validation
- Error messages
- Banking Example: Transaction validation rules

#### **File Q29**: API Integration - React Query
- useQuery and useMutation
- Cache management
- Optimistic updates
- Pagination and infinite queries
- Banking Example: Transaction history with caching

#### **File Q30**: React Query Advanced
- Query invalidation
- Prefetching data
- Parallel and dependent queries
- Error handling
- Banking Example: Real-time account data synchronization

---

### 📗 **ADVANCED LEVEL** (Questions 31-40)

#### **File Q31**: Jest & Testing Basics
- Test structure and matchers
- Mocking functions and modules
- Snapshot testing
- Coverage reports
- Banking Example: Testing transaction validators

#### **File Q32**: React Testing Library
- render and screen queries
- userEvent vs fireEvent
- Testing async components
- Best practices
- Banking Example: Testing banking forms

#### **File Q33**: Component Testing Strategies
- Unit vs Integration tests
- Testing hooks
- Testing context providers
- Testing Redux-connected components
- Banking Example: Testing payment flow

#### **File Q34**: E2E Testing - Cypress
- Cypress setup and commands
- Testing user flows
- Intercepting network requests
- Visual testing
- Banking Example: Complete transaction E2E test

#### **File Q35**: Code Splitting & Lazy Loading
- React.lazy and Suspense
- Route-based code splitting
- Component-based splitting
- Bundle analysis
- Banking Example: Lazy loading banking modules

#### **File Q36**: Performance Optimization
- React DevTools Profiler
- Identifying bottlenecks
- Virtualization (react-window)
- Debouncing and throttling
- Banking Example: Optimizing large transaction tables

#### **File Q37**: Server-Side Rendering (SSR)
- SSR concepts and benefits
- Next.js getServerSideProps
- Hydration process
- SEO optimization
- Banking Example: Public banking website with SSR

#### **File Q38**: Static Site Generation (SSG)
- getStaticProps and getStaticPaths
- Incremental Static Regeneration
- When to use SSG vs SSR
- Build-time optimization
- Banking Example: Banking blog and marketing pages

#### **File Q39**: Image Optimization
- Next.js Image component
- Responsive images
- Lazy loading strategies
- WebP and AVIF formats
- Banking Example: Optimized check images

#### **File Q40**: Web Vitals & Performance Metrics
- Core Web Vitals (LCP, FID, CLS)
- Measuring performance
- Lighthouse audits
- Real User Monitoring
- Banking Example: Monitoring banking app performance

---

### 🚀 **MICRO FRONTEND LEVEL** (Questions 41-50)

#### **File Q41**: Micro Frontend Architecture
- What are Micro Frontends?
- Architecture patterns and benefits
- When to use MFE vs Monolith
- Team autonomy and independent deployments
- Banking Example: Breaking banking app into micro frontends

#### **File Q42**: Module Federation - Webpack 5
- Module Federation concepts
- Host and Remote applications
- Shared dependencies
- Runtime integration
- Banking Example: Banking dashboard as host + remotes

#### **File Q43**: Module Federation Implementation
- webpack.config.js setup
- ModuleFederationPlugin configuration
- Exposing and consuming components
- TypeScript support
- Banking Example: Accounts, Payments, Transfers as remotes

#### **File Q44**: Single-SPA Framework
- Single-SPA architecture
- Root config and applications
- Lifecycle functions
- Framework agnostic approach
- Banking Example: Multi-framework banking portal

#### **File Q45**: Micro Frontend Communication
- Cross-app communication patterns
- Custom events
- Shared state management
- Props and callbacks
- Banking Example: Cart updates across micro frontends

#### **File Q46**: Routing in Micro Frontends
- Route ownership strategies
- Single-SPA routing
- Nested routing
- Cross-app navigation
- Banking Example: Banking app routing architecture

#### **File Q47**: Shared Dependencies & Libraries
- Dependency sharing strategies
- Singleton patterns
- Version management
- Shared UI component library
- Banking Example: Shared banking UI components

#### **File Q48**: Micro Frontend Deployment
- Independent deployment pipelines
- CI/CD for micro frontends
- Versioning strategies
- Rollback procedures
- Banking Example: Deploying banking remotes independently

#### **File Q49**: Micro Frontend Testing
- Contract testing
- Integration testing across MFEs
- E2E testing strategies
- Component testing
- Banking Example: Testing banking micro frontend ecosystem

#### **File Q50**: Production Best Practices
- Monitoring and observability
- Error handling across MFEs
- Performance optimization
- Security considerations
- Banking Example: Production-ready banking micro frontend architecture

---

## 📊 Progress Tracking

### Completed Files: 0/50 (0%)
- (None yet)

### In Progress: 0/50
- (None)

### Not Started: 50/50 (100%)
- ⏳ Q01-Q10 (Beginner Level)
- ⏳ Q11-Q20 (Intermediate Level)
- ⏳ Q21-Q30 (Intermediate-Advanced Level)
- ⏳ Q31-Q40 (Advanced Level)
- ⏳ Q41-Q50 (Micro Frontend Level) - **10 Questions**

---

## 🎯 Quality Standards

Each file must include:

1. **Summary Section**: What's covered in the question with key concepts
2. **Architecture Explanation**: Deep dive into React internals and design patterns
3. **Real-World Scenario**: Banking/E-commerce practical problem to solve
4. **3 Production-Ready Code Examples**:
   - Example 1: Basic implementation with TypeScript
   - Example 2: Advanced/production features with error handling
   - Example 3: Edge cases or alternative approaches
5. **Testing Instructions**: Actual commands and test examples
6. **Performance Analysis**: Comparisons, profiling, and optimization
7. **DO's and DON'Ts**: Best practices (10 each)
8. **Key Takeaways**: Summary of learnings and interview tips

**Target Length**: 1,500-2,000 lines per file

---

## 🚀 Development Approach

1. **Create file** with initial structure
2. **Add content in chunks**:
   - Chunk 1: Architecture explanation + Scenario
   - Chunk 2: Example 1 (Basic with TypeScript)
   - Chunk 3: Example 2 (Production-ready)
   - Chunk 4: Example 3 (Edge cases)
   - Chunk 5: Testing + Performance analysis
   - Chunk 6: DO's/DON'Ts + Takeaways
3. **Review and test** code examples
4. **Move to next file**

---

## 🏗️ Micro Frontend Architecture Overview (Q41-Q50)

### Banking Application Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│              Banking App - Micro Frontend Structure                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Host Application (Shell - Port 3000)                               │
│  ├─ Navigation & Layout                                             │
│  ├─ Authentication & Authorization                                  │
│  ├─ Shared Context (User, Theme)                                    │
│  └─ Route Configuration                                             │
│                                                                      │
│  Remote: Accounts Module (Port 3001)                                │
│  ├─ Account Dashboard                                               │
│  ├─ Account Details                                                 │
│  ├─ Transaction History                                             │
│  └─ Statements                                                      │
│                                                                      │
│  Remote: Payments Module (Port 3002)                                │
│  ├─ Bill Payments                                                   │
│  ├─ Credit Card Payments                                            │
│  ├─ P2P Transfers                                                   │
│  └─ Payment History                                                 │
│                                                                      │
│  Remote: Transfers Module (Port 3003)                               │
│  ├─ Internal Transfers                                              │
│  ├─ External Transfers                                              │
│  ├─ Beneficiary Management                                          │
│  └─ Scheduled Transfers                                             │
│                                                                      │
│  Remote: Investments Module (Port 3004)                             │
│  ├─ Portfolio Dashboard                                             │
│  ├─ Buy/Sell Stocks                                                 │
│  ├─ Investment Analysis                                             │
│  └─ Market Data                                                     │
│                                                                      │
│  Shared Libraries (NPM Packages)                                    │
│  ├─ UI Components (@bank/ui-components)                             │
│  ├─ Utils (@bank/utils)                                             │
│  ├─ Types (@bank/types)                                             │
│  └─ API Client (@bank/api-client)                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Technologies for Micro Frontends (Q41-Q50)
- **Webpack 5 Module Federation** - Primary MFE solution
- **Single-SPA** - Framework-agnostic orchestration
- **React 18** - Concurrent features and Suspense
- **TypeScript** - Type safety across remotes
- **Nx/Lerna** - Monorepo management
- **React Router** - Cross-app routing
- **React Query** - Shared data fetching
- **Jest + Cypress** - Testing strategies

---

## 📝 Notes

- All examples must use **banking/e-commerce context**
- Code must be **production-ready** with TypeScript and error handling
- Include **actual testing commands** that can be run
- Focus on **real-world problems** faced by enterprise applications
- **Last 10 questions (Q41-Q50)** deeply cover Micro Frontend Architecture
- Maintain **consistent quality** across all 50 files

---

**Last Updated**: November 30, 2025  
**Status**: ✅ Master Plan Ready - Focused on Question/Answer format with Micro Frontends


