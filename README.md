🧱 Requirements:
Make sure you include the SweetAlert2 and a spinner overlay.

👉 1. Add SweetAlert2 (in layout or main view)
html
Copy
Edit
<link href="https://cdn.jsdelivr.net/npm/sweetalert2@11/dist/sweetalert2.min.css" rel="stylesheet" />
<script src="https://cdn.jsdelivr.net/npm/sweetalert2@11"></script>
👉 2. Add loading spinner overlay (in main view)
html
Copy
Edit
<div id="loadingOverlay" style="display:none; position:fixed; top:0; left:0; width:100%; height:100%; background-color:rgba(255,255,255,0.7); z-index:9999; text-align:center;">
    <div style="position:absolute; top:50%; left:50%; transform:translate(-50%, -50%);">
        <div class="spinner-border text-primary" role="status" style="width: 3rem; height: 3rem;">
            <span class="visually-hidden">Loading...</span>
        </div>
        <p>Processing...</p>
    </div>
</div>
✅
