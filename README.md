# Living Codebase - 3D Repository Visualization Platform

## Project Overview
Build a 3D interactive visualization tool that transforms GitHub repositories into explorable force-directed graphs, highlighting "hot zones" (frequently modified/buggy code) through visual analytics.

## Tech Stack
- **Frontend**: React 18 + TypeScript + Vite
- **3D Rendering**: Three.js + React Three Fiber + Drei
- **State Management**: Zustand
- **Styling**: TailwindCSS
- **Physics Simulation**: D3-force-3d (Web Worker)
- **API Integration**: GitHub REST API + Octokit

---

## Project Structure

```
living-codebase/
├── src/
│   ├── components/
│   │   ├── canvas/
│   │   │   ├── Scene.tsx                 # Main 3D scene container
│   │   │   ├── GraphNode.tsx             # Individual file/folder nodes
│   │   │   ├── GraphEdge.tsx             # Dependency connections
│   │   │   ├── Lighting.tsx              # Scene lighting setup
│   │   │   └── CameraControls.tsx        # Navigation controls
│   │   ├── ui/
│   │   │   ├── FileDetailsPanel.tsx      # Selected node info
│   │   │   ├── SearchBar.tsx             # Fuzzy file search
│   │   │   ├── FilterControls.tsx        # Type/heat filters
│   │   │   ├── Legend.tsx                # Color key
│   │   │   └── LoadingState.tsx          # Loading indicators
│   │   └── layout/
│   │       ├── AppLayout.tsx             # Main app wrapper
│   │       └── Sidebar.tsx               # Settings sidebar
│   ├── lib/
│   │   ├── github/
│   │   │   ├── api.ts                    # GitHub API client
│   │   │   ├── auth.ts                   # Authentication flow
│   │   │   ├── fetchers.ts               # Data fetching functions
│   │   │   └── cache.ts                  # Response caching
│   │   ├── parser/
│   │   │   ├── imports.ts                # Import statement parser
│   │   │   ├── resolver.ts               # Path resolution
│   │   │   └── graph-builder.ts          # Dependency graph construction
│   │   ├── simulation/
│   │   │   ├── forces.ts                 # Physics force functions
│   │   │   ├── worker.ts                 # Web Worker controller
│   │   │   └── integration.ts            # D3-force-3d wrapper
│   │   └── analysis/
│   │       ├── hot-zones.ts              # Commit frequency analysis
│   │       ├── metrics.ts                # Metric calculations
│   │       └── scoring.ts                # Heat score algorithm
│   ├── stores/
│   │   ├── repoStore.ts                  # Repository data
│   │   ├── simulationStore.ts            # Physics state
│   │   ├── uiStore.ts                    # UI state
│   │   └── settingsStore.ts              # User preferences
│   ├── hooks/
│   │   ├── useSimulation.ts              # Simulation lifecycle
│   │   ├── useGitHubData.ts              # Data fetching
│   │   ├── useNodeInteraction.ts         # Node hover/click
│   │   └── useCameraAnimation.ts         # Camera transitions
│   ├── types/
│   │   ├── graph.ts                      # Graph data structures
│   │   ├── github.ts                     # GitHub API types
│   │   └── simulation.ts                 # Physics types
│   └── utils/
│       ├── colors.ts                     # Color schemes
│       ├── math.ts                       # Vector math helpers
│       └── mock-data.ts                  # Testing data generator
├── workers/
│   └── simulation.worker.ts              # Physics Web Worker
└── public/

```

---

## Type Definitions (src/types/graph.ts)

```typescript
export interface GraphNode {
  id: string;
  path: string;
  name: string;
  type: 'file' | 'folder';
  extension?: string;
  size: number;
  lastModified: Date;
  
  // Hot zone metrics
  commitCount: number;
  bugFixCount: number;
  recentCommits: number; // Last 30 days
  contributorCount: number;
  heatScore: number; // 0-1 normalized
  
  // Simulation properties
  position: [number, number, number];
  velocity: [number, number, number];
  fixed?: boolean;
  
  // Graph properties
  dependencies: string[]; // IDs of nodes this depends on
  dependents: string[]; // IDs of nodes that depend on this
}

export interface GraphEdge {
  id: string;
  source: string; // Node ID
  target: string; // Node ID
  type: 'dependency' | 'parent-child';
  strength: number; // 0-1 for visual weight
}

export interface RepoData {
  owner: string;
  name: string;
  nodes: Map<string, GraphNode>;
  edges: GraphEdge[];
  rootPath: string;
}

export interface HotZoneMetrics {
  commitFrequency: number;
  bugDensity: number;
  recencyScore: number;
  churnRate: number;
}
```

---

## Implementation Steps

### Step 1: Project Initialization

```bash
npm create vite@latest living-codebase -- --template react-ts
cd living-codebase
npm install three @react-three/fiber @react-three/drei
npm install zustand
npm install d3-force-3d
npm install @octokit/rest
npm install -D tailwindcss postcss autoprefixer @types/three
npx tailwindcss init -p
```

**Configure `tailwind.config.js`:**
```javascript
export default {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  theme: { extend: {} },
  plugins: [],
}
```

---

### Step 2: Zustand Stores

**src/stores/repoStore.ts:**
```typescript
import { create } from 'zustand';
import { RepoData, GraphNode } from '../types/graph';

interface RepoStore {
  repoData: RepoData | null;
  loading: boolean;
  error: string | null;
  setRepoData: (data: RepoData) => void;
  setLoading: (loading: boolean) => void;
  setError: (error: string | null) => void;
  getNode: (id: string) => GraphNode | undefined;
}

export const useRepoStore = create<RepoStore>((set, get) => ({
  repoData: null,
  loading: false,
  error: null,
  setRepoData: (data) => set({ repoData: data }),
  setLoading: (loading) => set({ loading }),
  setError: (error) => set({ error }),
  getNode: (id) => get().repoData?.nodes.get(id),
}));
```

**src/stores/simulationStore.ts:**
```typescript
import { create } from 'zustand';

interface SimulationStore {
  running: boolean;
  nodePositions: Map<string, [number, number, number]>;
  alpha: number; // Simulation heat
  toggleSimulation: () => void;
  updatePositions: (positions: Map<string, [number, number, number]>) => void;
  reset: () => void;
}

export const useSimulationStore = create<SimulationStore>((set) => ({
  running: false,
  nodePositions: new Map(),
  alpha: 1,
  toggleSimulation: () => set((state) => ({ running: !state.running })),
  updatePositions: (positions) => set({ nodePositions: positions }),
  reset: () => set({ nodePositions: new Map(), alpha: 1 }),
}));
```

**src/stores/uiStore.ts:**
```typescript
import { create } from 'zustand';

interface UIStore {
  selectedNodeId: string | null;
  hoveredNodeId: string | null;
  searchQuery: string;
  filters: {
    fileTypes: string[];
    minHeatScore: number;
  };
  setSelectedNode: (id: string | null) => void;
  setHoveredNode: (id: string | null) => void;
  setSearchQuery: (query: string) => void;
  setFilters: (filters: Partial<UIStore['filters']>) => void;
}

export const useUIStore = create<UIStore>((set) => ({
  selectedNodeId: null,
  hoveredNodeId: null,
  searchQuery: '',
  filters: { fileTypes: [], minHeatScore: 0 },
  setSelectedNode: (id) => set({ selectedNodeId: id }),
  setHoveredNode: (id) => set({ hoveredNodeId: id }),
  setSearchQuery: (query) => set({ searchQuery: query }),
  setFilters: (filters) => set((state) => ({ 
    filters: { ...state.filters, ...filters } 
  })),
}));
```

---

### Step 3: Mock Data Generator (for testing)

**src/utils/mock-data.ts:**
```typescript
import { RepoData, GraphNode, GraphEdge } from '../types/graph';

export function generateMockRepo(): RepoData {
  const nodes = new Map<string, GraphNode>();
  const edges: GraphEdge[] = [];

  // Create some sample files
  const files = [
    { path: 'src/index.ts', deps: ['src/App.tsx'] },
    { path: 'src/App.tsx', deps: ['src/components/Header.tsx', 'src/utils/helpers.ts'] },
    { path: 'src/components/Header.tsx', deps: [] },
    { path: 'src/utils/helpers.ts', deps: [] },
    { path: 'src/stores/userStore.ts', deps: [] },
  ];

  files.forEach((file, i) => {
    const node: GraphNode = {
      id: file.path,
      path: file.path,
      name: file.path.split('/').pop()!,
      type: 'file',
      extension: file.path.split('.').pop(),
      size: Math.random() * 10000,
      lastModified: new Date(Date.now() - Math.random() * 90 * 24 * 60 * 60 * 1000),
      commitCount: Math.floor(Math.random() * 50),
      bugFixCount: Math.floor(Math.random() * 10),
      recentCommits: Math.floor(Math.random() * 5),
      contributorCount: Math.floor(Math.random() * 5) + 1,
      heatScore: Math.random(),
      position: [
        (Math.random() - 0.5) * 100,
        (Math.random() - 0.5) * 100,
        (Math.random() - 0.5) * 100,
      ],
      velocity: [0, 0, 0],
      dependencies: file.deps,
      dependents: [],
    };
    nodes.set(node.id, node);

    // Create edges
    file.deps.forEach((depPath) => {
      edges.push({
        id: `${file.path}->${depPath}`,
        source: file.path,
        target: depPath,
        type: 'dependency',
        strength: 0.5 + Math.random() * 0.5,
      });
    });
  });

  return {
    owner: 'mockuser',
    name: 'mock-repo',
    nodes,
    edges,
    rootPath: 'src',
  };
}
```

---

### Step 4: Basic 3D Scene

**src/components/canvas/Scene.tsx:**
```typescript
import { Canvas } from '@react-three/fiber';
import { OrbitControls, Grid } from '@react-three/drei';
import GraphNode from './GraphNode';
import GraphEdge from './GraphEdge';
import { useRepoStore } from '../../stores/repoStore';

export default function Scene() {
  const repoData = useRepoStore((state) => state.repoData);

  if (!repoData) return null;

  return (
    <Canvas camera={{ position: [0, 50, 100], fov: 60 }}>
      {/* Lighting */}
      <ambientLight intensity={0.4} />
      <directionalLight position={[10, 10, 5]} intensity={0.8} />
      <pointLight position={[-10, -10, -5]} intensity={0.4} />

      {/* Controls */}
      <OrbitControls makeDefault />

      {/* Grid helper */}
      <Grid args={[100, 100]} />

      {/* Render edges first (so they appear behind nodes) */}
      {repoData.edges.map((edge) => (
        <GraphEdge key={edge.id} edge={edge} />
      ))}

      {/* Render nodes */}
      {Array.from(repoData.nodes.values()).map((node) => (
        <GraphNode key={node.id} node={node} />
      ))}
    </Canvas>
  );
}
```

---

### Step 5: Node Rendering

**src/components/canvas/GraphNode.tsx:**
```typescript
import { useRef, useState } from 'react';
import { Sphere } from '@react-three/drei';
import { GraphNode as GraphNodeType } from '../../types/graph';
import { useUIStore } from '../../stores/uiStore';
import * as THREE from 'three';

interface Props {
  node: GraphNodeType;
}

export default function GraphNode({ node }: Props) {
  const meshRef = useRef<THREE.Mesh>(null);
  const [hovered, setHovered] = useState(false);
  const setSelectedNode = useUIStore((state) => state.setSelectedNode);

  // Color based on heat score (blue -> yellow -> red)
  const color = new THREE.Color().setHSL(
    (1 - node.heatScore) * 0.6, // Hue: 0.6 (blue) to 0 (red)
    0.8,
    0.5
  );

  // Size based on importance (commit count + dependencies)
  const size = 1 + (node.commitCount / 50) * 2 + node.dependencies.length * 0.5;

  return (
    <Sphere
      ref={meshRef}
      position={node.position}
      args={[size, 16, 16]}
      onClick={() => setSelectedNode(node.id)}
      onPointerOver={() => setHovered(true)}
      onPointerOut={() => setHovered(false)}
    >
      <meshStandardMaterial
        color={color}
        emissive={hovered ? color : new THREE.Color(0x000000)}
        emissiveIntensity={hovered ? 0.3 : 0}
      />
    </Sphere>
  );
}
```

---

### Step 6: Edge Rendering

**src/components/canvas/GraphEdge.tsx:**
```typescript
import { useMemo } from 'react';
import { Line } from '@react-three/drei';
import { GraphEdge as GraphEdgeType } from '../../types/graph';
import { useRepoStore } from '../../stores/repoStore';
import * as THREE from 'three';

interface Props {
  edge: GraphEdgeType;
}

export default function GraphEdge({ edge }: Props) {
  const getNode = useRepoStore((state) => state.getNode);

  const sourceNode = getNode(edge.source);
  const targetNode = getNode(edge.target);

  const points = useMemo(() => {
    if (!sourceNode || !targetNode) return [];
    return [
      new THREE.Vector3(...sourceNode.position),
      new THREE.Vector3(...targetNode.position),
    ];
  }, [sourceNode, targetNode]);

  if (points.length === 0) return null;

  return (
    <Line
      points={points}
      color="#4a5568"
      lineWidth={edge.strength * 2}
      transparent
      opacity={0.3}
    />
  );
}
```

---

### Step 7: GitHub API Integration

**src/lib/github/api.ts:**
```typescript
import { Octokit } from '@octokit/rest';

export class GitHubClient {
  private octokit: Octokit;

  constructor(token: string) {
    this.octokit = new Octokit({ auth: token });
  }

  async getRepoTree(owner: string, repo: string, branch = 'main') {
    const { data } = await this.octokit.git.getTree({
      owner,
      repo,
      tree_sha: branch,
      recursive: 'true',
    });
    return data.tree;
  }

  async getCommitHistory(owner: string, repo: string, path?: string) {
    const { data } = await this.octokit.repos.listCommits({
      owner,
      repo,
      path,
      per_page: 100,
    });
    return data;
  }

  async getFileContent(owner: string, repo: string, path: string) {
    const { data } = await this.octokit.repos.getContent({
      owner,
      repo,
      path,
    });
    
    if ('content' in data) {
      return Buffer.from(data.content, 'base64').toString('utf-8');
    }
    throw new Error('Not a file');
  }
}
```

---

### Step 8: Dependency Parser

**src/lib/parser/imports.ts:**
```typescript
export function parseImports(code: string): string[] {
  const imports: string[] = [];
  
  // ES6 imports: import ... from '...'
  const es6Regex = /import\s+(?:[\w\s{},*]+\s+from\s+)?['"]([^'"]+)['"]/g;
  let match;
  while ((match = es6Regex.exec(code)) !== null) {
    imports.push(match[1]);
  }
  
  // CommonJS: require('...')
  const cjsRegex = /require\s*\(\s*['"]([^'"]+)['"]\s*\)/g;
  while ((match = cjsRegex.exec(code)) !== null) {
    imports.push(match[1]);
  }
  
  return imports.filter(imp => imp.startsWith('./') || imp.startsWith('../'));
}

export function resolveImportPath(fromPath: string, importPath: string): string {
  const fromDir = fromPath.split('/').slice(0, -1).join('/');
  const parts = importPath.split('/');
  const resolved = fromDir.split('/');
  
  for (const part of parts) {
    if (part === '..') resolved.pop();
    else if (part !== '.') resolved.push(part);
  }
  
  let fullPath = resolved.join('/');
  
  // Add common extensions if missing
  if (!/\.(ts|tsx|js|jsx)$/.test(fullPath)) {
    for (const ext of ['.ts', '.tsx', '.js', '.jsx']) {
      // In real implementation, check if file exists
      return fullPath + ext;
    }
  }
  
  return fullPath;
}
```

---

### Step 9: Hot Zones Analysis

**src/lib/analysis/hot-zones.ts:**
```typescript
import { GraphNode } from '../../types/graph';

export function calculateHeatScore(node: GraphNode): number {
  // Normalize metrics to 0-1 scale
  const commitScore = Math.min(node.commitCount / 100, 1);
  const bugScore = Math.min(node.bugFixCount / 20, 1);
  const recencyScore = Math.min(node.recentCommits / 10, 1);
  const churnScore = Math.min(node.contributorCount / 10, 1);
  
  // Weighted combination
  const weights = {
    commits: 0.3,
    bugs: 0.4,
    recency: 0.2,
    churn: 0.1,
  };
  
  return (
    commitScore * weights.commits +
    bugScore * weights.bugs +
    recencyScore * weights.recency +
    churnScore * weights.churn
  );
}

export function detectBugFix(commitMessage: string): boolean {
  const keywords = ['fix', 'bug', 'patch', 'hotfix', 'resolve', 'issue'];
  const lowerMessage = commitMessage.toLowerCase();
  return keywords.some(keyword => lowerMessage.includes(keyword));
}
```

---

### Step 10: Web Worker for Physics (Optional but Recommended)

**workers/simulation.worker.ts:**
```typescript
import forceSimulation from 'd3-force-3d';

let simulation: any = null;

self.onmessage = (e) => {
  const { type, data } = e.data;
  
  if (type === 'START') {
    const { nodes, edges } = data;
    
    simulation = forceSimulation(nodes)
      .force('charge', forceSimulation.forceManyBody().strength(-100))
      .force('link', forceSimulation.forceLink(edges).distance(30))
      .force('center', forceSimulation.forceCenter(0, 0, 0))
      .on('tick', () => {
        const positions = new Map();
        nodes.forEach((node: any) => {
          positions.set(node.id, [node.x, node.y, node.z]);
        });
        self.postMessage({ type: 'TICK', positions });
      });
  }
  
  if (type === 'STOP') {
    simulation?.stop();
  }
};
```

---

### Step 11: Main App Component

**src/App.tsx:**
```typescript
import { useEffect } from 'react';
import Scene from './components/canvas/Scene';
import FileDetailsPanel from './components/ui/FileDetailsPanel';
import SearchBar from './components/ui/SearchBar';
import { useRepoStore } from './stores/repoStore';
import { generateMockRepo } from './utils/mock-data';

export default function App() {
  const setRepoData = useRepoStore((state) => state.setRepoData);

  useEffect(() => {
    // Load mock data for testing
    setRepoData(generateMockRepo());
  }, [setRepoData]);

  return (
    <div className="w-screen h-screen bg-gray-900 text-white">
      {/* Search overlay */}
      <div className="absolute top-4 left-1/2 -translate-x-1/2 z-10">
        <SearchBar />
      </div>

      {/* 3D Scene */}
      <Scene />

      {/* Details panel */}
      <FileDetailsPanel />
    </div>
  );
}
```

---

## Key Implementation Notes

### Performance Optimizations
1. **Instanced Rendering**: For 1000+ nodes, use `<Instances>` from drei
2. **Frustum Culling**: Three.js does this automatically
3. **LOD**: Show simplified geometry for distant nodes
4. **Web Worker**: Run physics in separate thread

### Camera Navigation
- Use `<CameraControls>` from drei for smooth transitions
- Implement "focus on node" with `camera.lookAt()` animation
- Add keyboard shortcuts (F to focus, Space to reset)

### GitHub API Rate Limits
- Cache responses in localStorage
- Use conditional requests with ETags
- Consider GitHub GraphQL API for efficiency

### Dependency Resolution Challenges
- Handle barrel exports (`index.ts`)
- Resolve TypeScript path aliases (`@/components`)
- Support `.js` importing `.ts` files
- Handle dynamic imports

### Hot Zone Visualization Ideas
- Particle effects on high-churn files
- Bloom post-processing for hot nodes
- Pulsing animation (vary scale over time)
- Trail effects showing recent changes

---

## Development Workflow

1. **Start with mock data** - Get visualization working first
2. **Add GitHub integration** - Connect to real repos
3. **Implement parser** - Build dependency graph
4. **Add hot zones** - Fetch commit history and analyze
5. **Optimize performance** - Profile with Chrome DevTools
6. **Polish UX** - Smooth animations, loading states

---

## Testing Repositories

Start with small repos to validate:
- Your own small projects (< 50 files)
- Popular libraries (React, Vue, Express)
- Monorepos (turborepo examples)

---

## Future Enhancements

- [ ] Multi-language support (Python, Java, Go)
- [ ] Real-time updates via GitHub webhooks
- [ ] Collaborative viewing (multiplayer)
- [ ] Time-travel (replay repo evolution)
- [ ] AR/VR support
- [ ] Export to formats (PNG, video, 3D model)

---

## Cursor-Specific Instructions

**When implementing:**
1. Follow the folder structure exactly as outlined
2. Use the provided type definitions before writing any components
3. Set up stores before building UI components
4. Test with mock data before integrating GitHub API
5. Implement Web Worker simulation after basic rendering works
6. Add UI overlays last

**Code style preferences:**
- Use functional components with hooks
- Prefer named exports for components
- Use TypeScript strict mode
- Keep components under 200 lines
- Extract reusable logic to custom hooks

**Dependencies versions to use:**
- react: ^18.3.1
- three: ^0.160.0
- @react-three/fiber: ^8.15.0
- @react-three/drei: ^9.90.0
- zustand: ^4.4.7
- d3-force-3d: ^3.0.5

---

## Common Issues & Solutions

**Issue**: Nodes not appearing
- Check camera position is far enough back
- Verify node positions are within view frustum
- Add ambient light to scene

**Issue**: Poor performance with many nodes
- Implement instanced rendering
- Use `<Instances>` from drei
- Reduce simulation tick rate

**Issue**: Import parsing fails
- Add more regex patterns for edge cases
- Use proper AST parser (Babel/TypeScript parser)
- Handle ES6, CommonJS, dynamic imports separately

**Issue**: GitHub API rate limits
- Implement exponential backoff
- Cache aggressively
- Use authenticated requests (5000 req/hr vs 60)

---

This markdown file contains everything needed to build the Living Codebase project from scratch. Feed this entire document to Cursor and ask it to implement each section systematically.