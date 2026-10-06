using System.IO;
using BepInEx;
using BepInEx.Configuration;
using BepInEx.IL2CPP;
using UnhollowerRuntimeLib;
using UnityEngine;
using UnityEngine.UI;

// Requires Deobfuscation.cs in the same project (provides the PlayerList alias).
// Targets the older BepInEx 6 be.577 (Unhollower) used by the Crab Game pack.
// Put your PNG at: Crab Game\BepInEx\config\playerlist_bg.png   (press F6 in game to reload it)
namespace PlayerListSkin
{
    [BepInPlugin("com.you.playerlistskin", "Player List Skin", "0.2.0")]
    public class Plugin : BasePlugin
    {
        internal static ConfigEntry<string> FileName;
        internal static ConfigEntry<string> RootName;
        internal static ConfigEntry<float> BgAlpha;
        internal static ConfigEntry<float> OtherAlpha;
        internal static ConfigEntry<float> SkipSmallerThan;
        internal static BepInEx.Logging.ManualLogSource L;

        public override void Load()
        {
            L = Log;

            FileName = Config.Bind("Skin", "FileName", "playerlist_bg.png",
                "PNG file in BepInEx\\config used as the panel background.");
            RootName = Config.Bind("Skin", "RootName", "WindowUI",
                "Name of the child under the PlayerList object that holds the window. Empty = whole PlayerList object.");
            BgAlpha = Config.Bind("Skin", "BackgroundAlpha", 1f,
                "Opacity of your PNG background (0 = invisible, 1 = solid).");
            OtherAlpha = Config.Bind("Skin", "OtherAlpha", 0.35f,
                "Opacity for every other panel image (black row, list area, scrollbar...). 1 = unchanged.");
            SkipSmallerThan = Config.Bind("Skin", "SkipSmallerThan", 40f,
                "Images smaller than this (in both width and height) are left alone, e.g. player avatars.");

            ClassInjector.RegisterTypeInIl2Cpp<SkinBehaviour>();

            GameObject go = new GameObject("PlayerListSkinRunner");
            UnityEngine.Object.DontDestroyOnLoad(go);
            go.hideFlags = HideFlags.HideAndDontSave;
            go.AddComponent<SkinBehaviour>();

            Log.LogInfo("Player List Skin loaded");
        }
    }

    public class SkinBehaviour : MonoBehaviour
    {
        public SkinBehaviour(System.IntPtr ptr) : base(ptr) { }

        Texture2D tex;
        bool triedLoad;
        float nextScan;
        int lastRootId;

        public void Update()
        {
            if (Input.GetKeyDown(KeyCode.F6))
            {
                triedLoad = false;
                lastRootId = 0;
            }
            if (!triedLoad) LoadTexture();

            if (Time.unscaledTime < nextScan) return;
            nextScan = Time.unscaledTime + 0.5f;

            PlayerList anchor = UnityEngine.Object.FindObjectOfType<PlayerList>();
            if (anchor == null) return;

            Transform root = FindRoot(anchor);
            if (root == null) return;

            if (root.GetInstanceID() != lastRootId)
            {
                lastRootId = root.GetInstanceID();
                Dump(root);
            }
            Apply(root);
        }

        void LoadTexture()
        {
            triedLoad = true;
            string path = Path.Combine(Paths.ConfigPath, Plugin.FileName.Value);
            if (!File.Exists(path))
            {
                Plugin.L.LogWarning("PNG not found, only transparency will be applied: " + path);
                return;
            }

            byte[] bytes = File.ReadAllBytes(path);
            Texture2D t = new Texture2D(2, 2, TextureFormat.RGBA32, false);
            if (!ImageConversion.LoadImage(t, bytes))
            {
                Plugin.L.LogWarning("Could not decode PNG: " + path);
                return;
            }
            t.hideFlags = HideFlags.HideAndDontSave;

            if (tex != null) UnityEngine.Object.Destroy(tex);
            tex = t;
            Plugin.L.LogInfo("Loaded PNG " + tex.width + "x" + tex.height);
        }

        Transform FindRoot(Component anchor)
        {
            Transform t = anchor.transform;
            string name = Plugin.RootName.Value;
            if (!string.IsNullOrEmpty(name))
            {
                Transform child = t.Find(name);
                if (child != null) return child;
                Plugin.L.LogWarning("Child '" + name + "' not found under " + t.name + ", using the whole object.");
            }
            return t;
        }

        // Crops the PNG so it fills the panel without being stretched.
        Rect CoverRect(Rect target)
        {
            if (tex == null || target.height <= 0f || tex.height <= 0) return new Rect(0f, 0f, 1f, 1f);
            float pa = target.width / target.height;
            float ta = (float)tex.width / tex.height;
            if (ta > pa)
            {
                float w = pa / ta;
                return new Rect((1f - w) * 0.5f, 0f, w, 1f);
            }
            float h = ta / pa;
            return new Rect(0f, (1f - h) * 0.5f, 1f, h);
        }

        static string PathOf(Transform t, Transform stopAt)
        {
            string s = t.name;
            while (t.parent != null && t != stopAt)
            {
                t = t.parent;
                s = t.name + "/" + s;
            }
            return s;
        }

        void Dump(Transform root)
        {
            Plugin.L.LogInfo("Panel root: " + PathOf(root, null) + "  size=" + root.GetComponent<RectTransform>().rect.size);

            foreach (RawImage r in root.GetComponentsInChildren<RawImage>(true))
            {
                Plugin.L.LogInfo("  RawImage: " + PathOf(r.transform, root) +
                                 "  size=" + r.rectTransform.rect.size + "  color=" + r.color);
            }
            foreach (Image img in root.GetComponentsInChildren<Image>(true))
            {
                Plugin.L.LogInfo("  Image: " + PathOf(img.transform, root) +
                                 "  size=" + img.rectTransform.rect.size + "  color=" + img.color);
            }
        }

        static bool IsTiny(Rect r)
        {
            float min = Plugin.SkipSmallerThan.Value;
            return r.width < min && r.height < min;
        }

        void Apply(Transform root)
        {
            var raws = root.GetComponentsInChildren<RawImage>(true);
            var imgs = root.GetComponentsInChildren<Image>(true);

            // The biggest RawImage is assumed to be the main blue/gray background.
            RawImage bg = null;
            float bestArea = 0f;
            foreach (RawImage r in raws)
            {
                Rect rc = r.rectTransform.rect;
                float area = rc.width * rc.height;
                if (area > bestArea) { bestArea = area; bg = r; }
            }

            foreach (RawImage r in raws)
            {
                if (r == bg)
                {
                    if (tex != null)
                    {
                        r.texture = tex;
                        r.uvRect = CoverRect(r.rectTransform.rect);
                        r.color = new Color(1f, 1f, 1f, Plugin.BgAlpha.Value);
                    }
                }
                else if (!IsTiny(r.rectTransform.rect))
                {
                    Color c = r.color;
                    c.a = Plugin.OtherAlpha.Value;
                    r.color = c;
                }
            }

            foreach (Image img in imgs)
            {
                if (IsTiny(img.rectTransform.rect)) continue;
                Color c = img.color;
                c.a = Plugin.OtherAlpha.Value;
                img.color = c;
            }
        }
    }
}
