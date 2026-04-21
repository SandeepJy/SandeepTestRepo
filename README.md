# SandeepTestRepo

```ruby
#!/usr/bin/env ruby
# Usage: FIGMA_TOKEN=xxx ruby figma_reduce.rb FILE_KEY NODE_ID [LIBRARY_FILE_KEY] [--md]
require 'net/http'
require 'json'
require 'uri'

TOKEN       = ENV.fetch('FIGMA_TOKEN')
FILE_KEY    = ARGV[0] or abort 'missing FILE_KEY'
NODE_ID_RAW = ARGV[1] or abort 'missing NODE_ID'
LIB_KEY     = (ARGV[2] && !ARGV[2].start_with?('--')) ? ARGV[2] : FILE_KEY
AS_MD       = ARGV.include?('--md')

# Figma URLs use hyphens (node-id=0-1); the REST API uses colons (0:1).
NODE_ID = NODE_ID_RAW.include?(':') ? NODE_ID_RAW : NODE_ID_RAW.tr('-', ':')

def get(path)
  uri = URI("https://api.figma.com#{path}")
  req = Net::HTTP::Get.new(uri); req['X-Figma-Token'] = TOKEN
  res = Net::HTTP.start(uri.hostname, uri.port, use_ssl: true) { |h| h.request(req) }
  JSON.parse(res.body)
end

def prune(h) = h.reject { |_, v| v.nil? || v == {} || v == [] }

# ---- fetch ---------------------------------------------------------------
resp = get("/v1/files/#{FILE_KEY}/nodes?ids=#{URI.encode_www_form_component(NODE_ID)}")
abort "Figma API: #{resp['err']}" if resp['err']

nodes = resp['nodes'] || {}
node_entry = nodes[NODE_ID]
node_entry = nodes.values.first if !node_entry && nodes.size == 1

doc = node_entry&.dig('document') or abort(
  "node #{NODE_ID} not found (requested as #{NODE_ID_RAW}; keys in response: #{nodes.keys.inspect})"
)

# /nodes nests components under each node; fall back to top-level if present.
components  = node_entry['components']    || resp['components']    || {}
comp_sets   = node_entry['componentSets'] || resp['componentSets'] || {}
styles      = node_entry['styles']        || resp['styles']        || {}

# variables (Enterprise only — ignore failures, we still have style fallback)
vars = {}
begin
  vresp = get("/v1/files/#{LIB_KEY}/variables/local")
  vcoll = vresp.dig('meta', 'variableCollections') || {}
  register = lambda do |alias_id, name|
    vars[alias_id] = name if alias_id && name
  end
  (vresp.dig('meta', 'variables') || {}).each do |id, v|
    coll = vcoll.dig(v['variableCollectionId'], 'name')
    full = [coll, v['name'].to_s.gsub('/', '.')].compact.join('.')
    register.(id, full)
    register.(v['id'], full) if v['id']
    register.(v['subscribed_id'], full) if v['subscribed_id']
  end
rescue StandardError
end

# ---- helpers (closures over the lookup maps) -----------------------------
component_name = ->(cid) {
  c = components[cid] or next nil
  c['componentSetId'] ? comp_sets.dig(c['componentSetId'], 'name') : c['name']
}

clean_props = ->(cp) { (cp || {}).to_h { |k, v| [k.split('#').first, v['value']] } }

resolve_var = lambda do |ref|
  vid = ref.is_a?(Hash) ? ref['id'] : nil
  next vid unless vid.is_a?(String)
  stripped = vid.sub(/\AVariableID:/, '')
  vars[vid] || vars[stripped] || (vars[stripped.split('/').last] if stripped.include?('/')) || vid
end

tokens_of = ->(n) {
  out = {}
  (n['boundVariables'] || {}).each do |k, v|
    out[k] = v.is_a?(Array) ? v.map(&resolve_var) : resolve_var.(v)
  end
  (n['styles'] || {}).each { |k, sid| out["style.#{k}"] = styles.dig(sid, 'name') }
  out.empty? ? nil : out
}

axis = ->(a) { a unless a.nil? || a == 'MIN' }

layout_of = ->(n) {
  return nil unless n['layoutMode'] && n['layoutMode'] != 'NONE'
  prune(
    direction:  n['layoutMode'] == 'VERTICAL' ? 'VStack' : 'HStack',
    wrap:       (true if n['layoutWrap'] == 'WRAP'),
    spacing:    (n['itemSpacing'] unless n['itemSpacing'].to_f.zero?),
    padding:    prune(t: n['paddingTop'], b: n['paddingBottom'],
                      l: n['paddingLeft'], r: n['paddingRight']),
    alignMain:  axis.(n['primaryAxisAlignItems']),
    alignCross: axis.(n['counterAxisAlignItems']),
    scroll:     (n['overflowDirection'] unless [nil, 'NONE'].include?(n['overflowDirection']))
  )
}

reduce = nil
reduce = ->(n) {
  case n['type']
  when 'INSTANCE'
    prune(
      component: component_name.(n['componentId']) || n['name'],
      id:        n['name'],
      props:     clean_props.(n['componentProperties']),
      tokens:    tokens_of.(n)
    )
  when 'TEXT'
    prune(type: 'Text', text: n['characters'], tokens: tokens_of.(n))
  else
    if n['children']
      kids = n['children'].reject { |c| c['visible'] == false }.map(&reduce).compact
      prune(type: 'Container', name: n['name'],
            layout: layout_of.(n), tokens: tokens_of.(n), children: kids)
    else
      { type: 'UNMAPPED', name: n['name'], figmaType: n['type'] }
    end
  end
}

ir = { screen: doc['name'], nodeId: NODE_ID, tree: reduce.(doc) }

# ---- output --------------------------------------------------------------
if AS_MD
  emit = nil
  emit = ->(n, d = 0) {
    pad = '  ' * d
    if n[:component]
      p = (n[:props] || {}).map { |k, v| "#{k}=#{v.inspect}" }.join(', ')
      t = n[:tokens] ? "  [#{n[:tokens].map { |k, v| "#{k}:#{v}" }.join(', ')}]" : ''
      puts "#{pad}- #{n[:component]}(#{p})#{t}  // #{n[:id]}"
    elsif n[:type] == 'Text'
      puts "#{pad}- Text #{n[:text].inspect}  #{n[:tokens]&.values&.join(' ')}"
    elsif n[:type] == 'UNMAPPED'
      puts "#{pad}- ⚠️ UNMAPPED #{n[:figmaType]} '#{n[:name]}'"
    else
      l = n[:layout] || {}
      meta = [l[:direction], l[:spacing] && "spacing=#{l[:spacing]}",
              l[:padding] && "padding=#{l[:padding]}"].compact.join(' ')
      puts "#{pad}- #{meta.empty? ? 'Group' : meta}  // #{n[:name]}"
      (n[:children] || []).each { |c| emit.(c, d + 1) }
    end
  }
  puts "# #{ir[:screen]}  (figma #{ir[:nodeId]})\n\n"
  emit.(ir[:tree])
else
  puts JSON.pretty_generate(ir)
end
```


```sh
#!/usr/bin/env bash
set -euo pipefail

# ---------------------------------------------------------------
# Usage:
#   FIGMA_TOKEN=figd_xxx ./figma-reduce.sh <FILE_KEY> <NODE_ID> [LIBRARY_FILE_KEY]
#
# LIBRARY_FILE_KEY is optional — used to resolve variables if your
# tokens live in a separate library file. Falls back to FILE_KEY.
# ---------------------------------------------------------------

FILE_KEY="${1:?file key required}"
NODE_ID="${2:?node id required}"
LIB_KEY="${3:-$FILE_KEY}"
: "${FIGMA_TOKEN:?FIGMA_TOKEN env var required}"

API="https://api.figma.com/v1"
HDR=(-sS -H "X-Figma-Token: $FIGMA_TOKEN")

TMP="$(mktemp -d)"
trap 'rm -rf "$TMP"' EXIT

# ---------------------------------------------------------------
# 1. Fetch the frame
# ---------------------------------------------------------------
>&2 echo "→ fetching frame $NODE_ID"
curl "${HDR[@]}" \
  "$API/files/$FILE_KEY/nodes?ids=$(jq -rn --arg n "$NODE_ID" '$n|@uri')" \
  > "$TMP/frame.json"

# ---------------------------------------------------------------
# 2. Fetch variables (Enterprise). Swallow 403 if not available.
# ---------------------------------------------------------------
>&2 echo "→ fetching variables (may 403 on non-Enterprise — that's OK)"
curl "${HDR[@]}" "$API/files/$LIB_KEY/variables/local" 2>/dev/null \
  | jq -e '.meta.variables // {}' > "$TMP/vars.json" 2>/dev/null \
  || echo '{}' > "$TMP/vars.json"

# ---------------------------------------------------------------
# 3. Build lookup maps with jq, then reduce the tree
# ---------------------------------------------------------------
jq -n \
  --slurpfile frame "$TMP/frame.json" \
  --slurpfile vars  "$TMP/vars.json"  \
  --arg nodeId "$NODE_ID" \
'
# ----- inputs ------------------------------------------------
($frame[0])                                  as $f      |
($f.nodes[$nodeId].document)                 as $root   |
($f.components    // {})                     as $comps  |
($f.componentSets // {})                     as $sets   |
($f.styles        // {})                     as $styles |
($vars[0]         // {})                     as $varmap |

# ----- helpers -----------------------------------------------

# componentId -> "SUInputField"
def compName($id):
  ($comps[$id]) as $c
  | if $c == null then null
    elif $c.componentSetId then ($sets[$c.componentSetId].name // $c.name)
    else $c.name end;

# "Title#201:0" -> "Title", and {type,value} -> value
def cleanProps:
  with_entries({
    key:   (.key | split("#")[0]),
    value: .value.value
  });

# VariableID:xxx -> "color/brand/primary"  (falls back to id)
def varName($id): ($varmap[$id].name // $id);

# styles.fill / styles.text id -> "Typography/Body/M"
def styleName($id): ($styles[$id].name // $id);

# boundVariables block -> { fills: "color/brand/primary", ... }
def tokens:
  if . == null then null
  else with_entries(
    .value |=
      if type == "array" then map(varName(.id))
      elif type == "object" and has("id") then varName(.id)
      else . end
  ) end;

def compact: with_entries(select(
  .value != null and .value != {} and .value != [] and .value != false
));

def layout:
  if (.layoutMode // "NONE") == "NONE" then null
  else {
    direction: (if .layoutMode == "VERTICAL" then "VStack" else "HStack" end),
    wrap:      (if (.layoutWrap // "") == "WRAP" then true else null end),
    spacing:   (.itemSpacing | if . == 0 then null else . end),
    padding:   ({t:.paddingTop, b:.paddingBottom, l:.paddingLeft, r:.paddingRight}
                | with_entries(select(.value != 0 and .value != null))
                | if length == 0 then null else . end),
    alignMain:  (.primaryAxisAlignItems  | if . == "MIN" or . == null then null else . end),
    alignCross: (.counterAxisAlignItems  | if . == "MIN" or . == null then null else . end),
    scroll:     (.overflowDirection | if . == "NONE" or . == null then null else . end)
  } | compact end;

# ----- core recursion ----------------------------------------
def walk_node:
  if .visible == false then empty

  elif .type == "INSTANCE" then
    {
      component: (compName(.componentId) // .name),
      id:        .name,
      props:     ((.componentProperties // {}) | cleanProps),
      tokens:    (.boundVariables | tokens),
      textStyle: (if .styles.text then styleName(.styles.text) else null end)
    } | compact

  elif .type == "TEXT" then
    {
      type:   "Text",
      text:   .characters,
      style:  (if .styles.text then styleName(.styles.text) else null end),
      tokens: (.boundVariables | tokens)
    } | compact

  elif has("children") then
    {
      type:     "Container",
      name:     .name,
      layout:   layout,
      tokens:   (.boundVariables | tokens),
      fillStyle:(if .styles.fill then styleName(.styles.fill) else null end),
      children: [ .children[] | walk_node ]
    } | compact

  else
    { type: "UNMAPPED", name: .name, figmaType: .type }
  end;

# ----- output ------------------------------------------------
{
  screen: $root.name,
  nodeId: $nodeId,
  tree:   ($root | walk_node)
}
' > "$TMP/ir.json"

# ---------------------------------------------------------------
# 4. Emit JSON
# ---------------------------------------------------------------
SCREEN_NAME="$(jq -r '.screen' "$TMP/ir.json" | tr ' /' '__')"
cp "$TMP/ir.json" "${SCREEN_NAME}.ir.json"
>&2 echo "✓ wrote ${SCREEN_NAME}.ir.json"

# ---------------------------------------------------------------
# 5. Emit compact Markdown (agent-friendly)
# ---------------------------------------------------------------
jq -r '
def ind($d): "  " * $d;

def props2str:
  to_entries | map("\(.key)=\(.value|tojson)") | join(", ");

def md($d):
  if has("component") then
    "\(ind($d))- \(.component)(\((.props // {}) | props2str))"
    + (if .id then "  // \(.id)" else "" end)
    + (if .tokens then "  tokens=\(.tokens|tojson)" else "" end)
  elif .type == "Text" then
    "\(ind($d))- Text \"\(.text)\""
    + (if .style then "  style=\(.style)" else "" end)
  elif .type == "Container" then
    ([ "\(ind($d))- \(.layout.direction // "Group")"
       + (if .layout.spacing then " spacing=\(.layout.spacing)" else "" end)
       + (if .layout.padding then " padding=\(.layout.padding|tojson)" else "" end)
       + (if .fillStyle then " bg=\(.fillStyle)" else "" end)
       + "  // \(.name)"
     ] + [ .children[]? | md($d+1) ]) | join("\n")
  else
    "\(ind($d))- ⚠️ UNMAPPED \(.figmaType) \"\(.name)\""
  end;

"# \(.screen)  (figma \(.nodeId))\n\n" + (.tree | md(0))
' "$TMP/ir.json" > "${SCREEN_NAME}.ir.md"

>&2 echo "✓ wrote ${SCREEN_NAME}.ir.md"
cat "${SCREEN_NAME}.ir.md"
```



