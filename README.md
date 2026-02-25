####GUI COMPARATOR
import customtkinter as ctk
from customtkinter import filedialog
from line_handler import Line
from line_handler import parseur_cfd_dummy

class RibbonApp(ctk.CTk):
    def __init__(self):
        super().__init__()

        self.title("Application avec Bandeau - Workflow Hybride")
        self.geometry("1100x800")
        
        # GLOBAL VARIABLES
        self.pairs_en_attente = []
        self.l1_orphelins = []
        self.l2_orphelins = []
        self.choix_finaux = [] 

        # --- Variables de suivi de l'interface ---
        self.outil_actif = None
        self.nom_outil_actif = None # Pour savoir si on est dans le Matcher ou le Selector

        self.grid_rowconfigure(1, weight=1)
        self.grid_columnconfigure(0, weight=1)

        self.setup_ribbon()
        self.setup_workspace()

    def setup_ribbon(self):
        self.ribbon = ctk.CTkTabview(self, height=50, command=self.on_tab_change)
        self.ribbon.grid(row=0, column=0, padx=10, pady=(10, 0), sticky="ew")

        self.ribbon.add("Accueil")
        self.ribbon.add("Appairage")
        self.ribbon.add("Sélection")

        tab_accueil = self.ribbon.tab("Accueil")
        ctk.CTkButton(tab_accueil, text="Sauvegarder").pack(side="left", padx=10, pady=5)
        ctk.CTkButton(tab_accueil, text="Charger données", command=self.charger_donnees).pack(side="left", padx=10, pady=5)

    def setup_workspace(self):
        self.workspace = ctk.CTkFrame(self)
        self.workspace.grid(row=1, column=0, padx=10, pady=10, sticky="nsew")
        self.workspace.grid_columnconfigure(0, weight=1)
        self.workspace.grid_rowconfigure(0, weight=1)
        
        self.afficher_accueil()

    def charger_donnees(self):    
        # Ouvre une fenêtre pour choisir les faux fichiers (tu peux choisir n'importe quel txt)
        chemin_v1 = filedialog.askopenfilename(title="Sélectionner ancien param.txt")
        if not chemin_v1: return

        chemin_v2 = filedialog.askopenfilename(title="Sélectionner nouveau template param.txt")
        if not chemin_v2: return

        # Appel du Mock
        appairages_auto, orph_v1, orph_v2, n = parseur_cfd_dummy(chemin_v1, chemin_v2)

        # Injection dans les listes du Hub
        self.l1_orphelins = orph_v1
        self.l2_orphelins = orph_v2
        
        self.pairs_en_attente = []
        for i, paire in enumerate(appairages_auto):
            # Pré-cochage des 'n' premiers éléments (0 = coche à gauche, 1 = coche à droite)
            choix = 0 if i < n else None
            self.pairs_en_attente.append((paire, choix))

        # On retourne sur l'accueil pour voir les compteurs mis à jour
        self.afficher_accueil()

    def on_tab_change(self):
        # 1. SAUVEGARDE AUTOMATIQUE 
        if self.outil_actif:
            if self.nom_outil_actif == "Appairage":
                self.outil_actif.valider()
            elif self.nom_outil_actif == "Sélection":
                # Si on quitte le sélecteur, on force la sauvegarde silencieuse
                self.outil_actif.valider(is_auto_save=True) 

        # On nettoie la mémoire AVANT de changer d'onglet
        self.outil_actif = None
        self.nom_outil_actif = None

        # 2. AFFICHAGE DU NOUVEL ONGLET
        onglet_actif = self.ribbon.get()
        if onglet_actif == "Accueil":
            self.afficher_accueil()
        elif onglet_actif == "Appairage":
            self.afficher_outil_appairage()
        elif onglet_actif == "Sélection":
            self.afficher_outil_selection()

    # --- Nettoyage ---
    def clear_workspace(self):
        for widget in self.workspace.winfo_children():
            widget.destroy()

    # --- ACCUEIL ---
    def afficher_accueil(self):
        self.clear_workspace()
        msg = f"Bienvenue.\n\nPaires en attente : {len(self.pairs_en_attente)}\nÉléments isolés : {len(self.l1_orphelins)} (G) / {len(self.l2_orphelins)} (D)\nChoix validés : {len(self.choix_finaux)}"
        ctk.CTkLabel(self.workspace, text=msg, font=ctk.CTkFont(size=20)).pack(expand=True)

    # --- APPAIRAGE (Sauvegarde Auto) ---
    def afficher_outil_appairage(self):
        self.clear_workspace()
        try:
            from matcher import MatcherFrame
            outil = MatcherFrame(master=self.workspace, 
                                 l1=self.l1_orphelins, 
                                 l2=self.l2_orphelins, 
                                 on_complete=self.recevoir_choix_appairage)
            outil.pack(fill="both", expand=True)
            
            # On indique au Hub que l'outil d'appairage est actif
            self.outil_actif = outil
            self.nom_outil_actif = "Appairage"
        except ImportError:
            ctk.CTkLabel(self.workspace, text="Erreur de chargement matcher.py").pack(expand=True)

    def recevoir_choix_appairage(self, nouvelles_paires, orphelins_l1_restants, orphelins_l2_restants):
        """Met à jour les données silencieusement (appelé par on_tab_change)."""
        self.pairs_en_attente.extend([(p, None) for p in nouvelles_paires])
        self.l1_orphelins = orphelins_l1_restants
        self.l2_orphelins = orphelins_l2_restants

    # --- SÉLECTION (Validation par Bouton) ---
    def afficher_outil_selection(self):
        self.clear_workspace()
        try:
            from selector import SelectorFrame 
            outil = SelectorFrame(master=self.workspace, 
                                  pairs=self.pairs_en_attente,
                                  on_complete=self.recevoir_choix_selection)
            outil.pack(fill="both", expand=True)
            
            # On indique au Hub que c'est le Sélecteur (il n'y aura pas de sauvegarde auto)
            self.outil_actif = outil
            self.nom_outil_actif = "Sélection"
        except ImportError:
            ctk.CTkLabel(self.workspace, text="Erreur de chargement selector.py").pack(expand=True)


    def recevoir_choix_selection(self, choix_valides, paires_restantes, paires_supprimees, is_auto_save):
        # 1. Mise à jour de toutes les listes
        self.choix_finaux.extend(choix_valides)
        self.pairs_en_attente = paires_restantes
        
        for element_gauche, element_droite in paires_supprimees:
            self.l1_orphelins.append(element_gauche)
            self.l2_orphelins.append(element_droite)

        # Si c'est l'utilisateur qui a cliqué sur le bouton "Valider" (donc PAS un auto-save)
        if not is_auto_save:
            # On efface la mémoire pour éviter une double-sauvegarde au changement d'onglet
            self.outil_actif = None 
            self.nom_outil_actif = None
            
            # On force le ruban à retourner sur l'accueil
            self.ribbon.set("Accueil")
            self.afficher_accueil()

if __name__ == "__main__":
    ctk.set_appearance_mode("system")
    app = RibbonApp()
    app.mainloop()

### MATCHER

import customtkinter as ctk
import difflib
import re # NOUVEAU : Module standard pour les Regex

# --- Fonction de distance ---
def distance_edition(line_a, line_b):
    ratio = difflib.SequenceMatcher(None, line_a.content, line_b.content).ratio()
    return 1.0 - ratio

# matcher.py

def distance_custom(line_a, line_b):
    """
    Utilise ta logique personnalisée.
    Ici, on simule l'appel à ta fonction sur les noms normalisés.
    """
    # Remplace le code ci-dessous par l'import de TA fonction
    import difflib
    ratio = difflib.SequenceMatcher(None, line_a.normalized, line_b.normalized).ratio()
    return 1.0 - ratio

# --- Pop-up 100% CustomTkinter ---
class CustomConfirmPopup(ctk.CTkToplevel):
    def __init__(self, parent, titre, message):
        super().__init__(parent)
        self.title(titre)
        self.result = False 
        
        self.transient(parent)
        self.grab_set()

        self.label = ctk.CTkLabel(self, text=message, font=ctk.CTkFont(size=15), justify="center")
        self.label.pack(pady=30, padx=20, fill="both", expand=True)
        
        btn_frame = ctk.CTkFrame(self, fg_color="transparent")
        btn_frame.pack(pady=(0, 20))
        
        ctk.CTkButton(btn_frame, text="OK", width=120, command=self.on_ok).pack(side="left", padx=15)
        ctk.CTkButton(btn_frame, text="Annuler", width=120, command=self.on_cancel).pack(side="right", padx=15)
        
        self.update_idletasks() 
        w, h = 400, 250
        p_width, p_height = parent.winfo_width(), parent.winfo_height()
        p_x, p_y = parent.winfo_rootx(), parent.winfo_rooty()
        x = p_x + (p_width // 2) - (w // 2)
        y = p_y + (p_height // 2) - (h // 2)
        self.geometry(f"{w}x{h}+{x}+{y}")
        self.resizable(False, False)

    def on_ok(self):
        self.result = True
        self.destroy()
        
    def on_cancel(self):
        self.result = False
        self.destroy()

# --- Interface d'appairage ---
class MatcherFrame(ctk.CTkFrame):
    def __init__(self, master, l1, l2, on_complete):
        super().__init__(master, fg_color="transparent")
        
        ###COSMETICS
        self.default_border = ("gray60", "gray40")
        self.match_border = ("darkgreen", "lightgreen")
        
        
        self.on_complete = on_complete
        
        self.l1_items = l1.copy()
        self.l2_items = l2.copy()
        self.new_pairs = []

        # NOUVEAU : On garde en mémoire l'ordre actuel des éléments (pour combiner Tri et Filtre)
        self.current_order_l = self.l1_items.copy()
        self.current_order_r = self.l2_items.copy()

        self.selected_left = None  
        self.selected_right = None 
        
        self.l_buttons = {}
        self.r_buttons = {}

        self.grid_columnconfigure((0, 1), weight=1)
        # NOUVEAU : C'est la ligne 2 qui s'étire maintenant (car la ligne 1 a les barres de recherche)
        self.grid_rowconfigure(2, weight=1)

        self.setup_ui()

    def setup_ui(self):
        ctk.CTkLabel(self, text="Appairage Intelligent & Filtrage Regex", 
                     font=ctk.CTkFont(size=18, weight="bold")).grid(row=0, column=0, columnspan=2, pady=10)
        titre = ctk.CTkLabel(self, text="Appairage Intelligent & Filtrage Regex", 
                             font=ctk.CTkFont(size=18, weight="bold"))
        titre.grid(row=0, column=0, columnspan=2, pady=10)
        self.bind("<Button-1>", lambda e: self.focus_set())
        titre.bind("<Button-1>", lambda e: self.focus_set())
        self.entry_l = ctk.CTkEntry(self, placeholder_text="Regex Liste 1 (ex: ^A.*)")
        self.entry_l.grid(row=1, column=0, padx=20, pady=(0, 10), sticky="ew")
        # <KeyRelease> déclenche la fonction à chaque fois que tu tapes ou effaces une touche
        self.entry_l.bind("<KeyRelease>", lambda e: self.refresh_column("left"))

        self.entry_r = ctk.CTkEntry(self, placeholder_text="Regex Liste 2 (ex: \d+$)")
        self.entry_r.grid(row=1, column=1, padx=20, pady=(0, 10), sticky="ew")
        self.entry_r.bind("<KeyRelease>", lambda e: self.refresh_column("right"))

        # --- Ligne 2 - Colonnes (ScrollableFrames) ---
        self.scroll_l = ctk.CTkScrollableFrame(self, label_text="Liste 1 (Orphelins G)")
        self.scroll_l.grid(row=2, column=0, padx=20, pady=(0, 10), sticky="nsew")

        # Au lieu de faire des .pack() ici, on crée juste les boutons en mémoire
        for obj in self.l1_items:
            btn = ctk.CTkButton(self.scroll_l, text=obj.display(), fg_color="transparent", border_width=2, border_color=self.default_border)
            btn.configure(command=lambda o=obj, b=btn: self.select_left(o, b))
            self.l_buttons[obj] = btn

        self.scroll_r = ctk.CTkScrollableFrame(self, label_text="Liste 2 (Orphelins D)")
        self.scroll_r.grid(row=2, column=1, padx=20, pady=(0, 10), sticky="nsew")

        for obj in self.l2_items:
            btn = ctk.CTkButton(self.scroll_r, text=obj.display(), fg_color="transparent", border_width=2, border_color=self.default_border)
            btn.configure(command=lambda o=obj, b=btn: self.select_right(o, b))
            self.r_buttons[obj] = btn

        # On appelle notre fonction d'affichage qui va trier, filtrer et placer (.pack) les éléments
        self.refresh_column("left")
        self.refresh_column("right")

    # --- NOUVEAU : Le Chef d'Orchestre (Filtre + Affichage) ---
    def refresh_column(self, side):
        """Met à jour l'affichage d'une colonne en respectant l'ordre de tri ET le filtre Regex."""
        if side == "left":
            items, buttons, pattern = self.current_order_l, self.l_buttons, self.entry_l.get()
            base_list = self.l1_items
        else:
            items, buttons, pattern = self.current_order_r, self.r_buttons, self.entry_r.get()
            base_list = self.l2_items

        # 1. Tentative de compilation de la Regex
        try:
            # re.IGNORECASE permet de ne pas se soucier des majuscules/minuscules
            regex = re.compile(pattern, re.IGNORECASE) if pattern else None
        except re.error:
            # Si la regex est invalide (ex: l'utilisateur a tapé "[" mais pas encore "]"), 
            # on interrompt la fonction silencieusement pour ne pas faire bugger l'interface.
            return

        # 2. On masque TOUS les boutons de cette colonne de l'interface
        for btn in buttons.values():
            btn.pack_forget()

        # 3. On réaffiche uniquement ceux qui matchent, dans le bon ordre !
        for obj in items:
            # Vérification de sécurité : si l'objet a été appairé et n'existe plus
            if obj not in base_list: 
                continue
                
            # Si pas de regex, ou si la regex trouve une correspondance dans le texte
            if regex is None or regex.search(obj.display()):
                btn = buttons[obj]
                btn.pack(pady=5, padx=10, fill="x")
    
    
    def reset_column_formatting(self, side):
        """Nettoie les bordures vertes et les scores de distance d'une colonne."""
        if side == "right":
            for obj, btn in self.r_buttons.items():
                btn.configure(border_color=self.default_border, text=obj.display())
            self.refresh_column("right")
        else:
            for obj, btn in self.l_buttons.items():
                btn.configure(border_color=self.default_border, text=obj.display())
            self.refresh_column("left")


    # --- Logique de Sélection et de Tri ---
    def select_left(self, line_obj, btn_widget):
        if self.selected_left and self.selected_left[0] == line_obj:
            btn_widget.configure(fg_color="transparent")
            self.selected_left = None
            self.reset_column_formatting("right") # On nettoie la colonne d'en face
            return 

        if self.selected_left:
            self.selected_left[1].configure(fg_color="transparent")
            
        btn_widget.configure(fg_color=("gray75", "gray25"))
        self.selected_left = (line_obj, btn_widget)
        
        if not self.selected_right:
            self.sort_opposite_list(reference_obj=line_obj, target_side="right")
            
        self.check_pairing(last_clicked="left")

    def select_right(self, line_obj, btn_widget):
        if self.selected_right and self.selected_right[0] == line_obj:
            btn_widget.configure(fg_color="transparent")
            self.selected_right = None
            self.reset_column_formatting("left") # On nettoie la colonne d'en face
            return

        if self.selected_right:
            self.selected_right[1].configure(fg_color="transparent")
            
        btn_widget.configure(fg_color=("gray75", "gray25"))
        self.selected_right = (line_obj, btn_widget)
        
        if not self.selected_left:
            self.sort_opposite_list(reference_obj=line_obj, target_side="left")
            
        self.check_pairing(last_clicked="right")



    def sort_opposite_list(self, reference_obj, target_side):
        if target_side == "right":
            items_to_sort, buttons_dict, current_selection = self.l2_items, self.r_buttons, self.selected_right
        else:
            items_to_sort, buttons_dict, current_selection = self.l1_items, self.l_buttons, self.selected_left

        scored_items = [(distance_custom(reference_obj, obj), obj) for obj in items_to_sort]
        scored_items.sort(key=lambda x: x[0])
        
        # 1. Mise à jour de la mémoire de l'ordre
        if target_side == "right":
            self.current_order_r = [obj for score, obj in scored_items]
        else:
            self.current_order_l = [obj for score, obj in scored_items]
        
        # 2. Mise à jour des textes et des couleurs de bordure
        for score, obj in scored_items:
            btn = buttons_dict[obj]
            btn.configure(text=f"{obj.display()}  [Dist: {score:.2f}]")
            
            if not (current_selection and current_selection[0] == obj):
                if score < 0.2:
                    btn.configure(border_color=self.match_border)
                else:
                    btn.configure(border_color=self.default_border)
                    
        # 3. NOUVEAU : On redessine la colonne (ce qui applique l'ordre ET le filtre Regex)
        self.refresh_column(target_side)

    # --- Vérification et Validation ---
    def check_pairing(self, last_clicked):
        if self.selected_left and self.selected_right:
            obj_l = self.selected_left[0]
            obj_r = self.selected_right[0]

            msg = f"Voulez-vous appairer ces éléments ?\n\n• {obj_l.display()}\n• {obj_r.display()}"
            
            popup = CustomConfirmPopup(self, "Confirmation d'appairage", msg)
            self.wait_window(popup)

            if popup.result:
                self.new_pairs.append((obj_l, obj_r))
                
                # 1. On retire les éléments des données
                self.l1_items.remove(obj_l)
                self.l2_items.remove(obj_r)
                
                del self.l_buttons[obj_l]
                del self.r_buttons[obj_r]
                
                # 2. On détruit les deux boutons cliqués
                self.selected_left[1].destroy()
                self.selected_right[1].destroy()
                
                self.selected_left = None
                self.selected_right = None

                # 3. Réinitialisation des autres boutons ---
                self.selected_left = None
                self.selected_right = None

                self.reset_column_formatting("left")
                self.reset_column_formatting("right")
                self.refresh_column("left")
                self.refresh_column("right")
                # -----------------------------------------------------

            else:
                if last_clicked == "left":
                    self.selected_left[1].configure(fg_color="transparent")
                    self.selected_left = None
                else:
                    self.selected_right[1].configure(fg_color="transparent")
                    self.selected_right = None

    def valider(self):
        self.on_complete(self.new_pairs, self.l1_items, self.l2_items)


#### SELECTOR


import customtkinter as ctk
from matcher import CustomConfirmPopup
import re
from line_handler import Line

class SelectorFrame(ctk.CTkFrame):
    def __init__(self, master, pairs, on_complete):
        super().__init__(master, fg_color="transparent") 
        
        self.pairs = [p[0] for p in pairs]       # On extrait les paires
        self.choices = [p[1] for p in pairs]     # On extrait la mémoire des coches
        self.on_complete = on_complete
        
        self.highlighted_indices = set() 
        self.deleted_indices = set()
        self.last_clicked_idx = None
        self.hide_completed = False

        # NOUVEAU : État de la logique de filtre
        self.logic_mode = "OR" 

        self.row_frames = []
        self.checks_l = []
        self.checks_r = []

        self.grid_columnconfigure(0, weight=1)
        # NOUVEAU : C'est la ligne 2 (la zone défilante) qui s'étire maintenant
        self.grid_rowconfigure(2, weight=1)

        # Perte de focus au clic sur le fond
        self.bind("<Button-1>", lambda e: self.focus_set())

        self.setup_ui()

    def setup_ui(self):
        # --- Barre d'outils ---
        self.toolbar = ctk.CTkFrame(self)
        self.toolbar.bind("<Button-1>", lambda e: self.focus_set())
        self.toolbar.grid(row=0, column=0, pady=(0, 10), sticky="ew")
        
        ### BARRE REGEX
        self.regex_frame = ctk.CTkFrame(self, fg_color="transparent")
        self.regex_frame.grid(row=1, column=0, padx=10, pady=(0, 10), sticky="ew")
        self.regex_frame.grid_columnconfigure((0, 2), weight=1) # Les champs de texte s'étirent
        self.regex_frame.bind("<Button-1>", lambda e: self.focus_set())

        self.entry_l = ctk.CTkEntry(self.regex_frame, placeholder_text="Regex Colonne G (ex: ^A.*)")
        self.entry_l.grid(row=0, column=0, sticky="ew")
        self.entry_l.bind("<KeyRelease>", self.on_regex_change)

        self.btn_logic = ctk.CTkButton(self.regex_frame, text="OR", width=60, fg_color="#3498db", hover_color="#2980b9", command=self.cycle_logic)
        self.btn_logic.grid(row=0, column=1, padx=10)

        self.entry_r = ctk.CTkEntry(self.regex_frame, placeholder_text="Regex Colonne D (ex: \d+$)")
        self.entry_r.grid(row=0, column=2, sticky="ew")
        self.entry_r.bind("<KeyRelease>", self.on_regex_change)

        btn_container = ctk.CTkFrame(self.toolbar, fg_color="transparent")
        btn_container.pack(pady=10, padx=10, fill="x")
        btn_container.grid_columnconfigure((0, 1, 2, 3, 4), weight=1)
        ###

        ### BOUTONS
        ctk.CTkButton(btn_container, text="Tout G", command=lambda: self.global_bulk_action(0)).grid(row=0, column=0, padx=5, sticky="ew")
        ctk.CTkButton(btn_container, text="Tout D", command=lambda: self.global_bulk_action(1)).grid(row=0, column=1, padx=5, sticky="ew")
        ctk.CTkButton(btn_container, text="Inverser", command=self.invert_all_choices).grid(row=0, column=2, padx=5, sticky="ew")
        ctk.CTkButton(btn_container, text="Trier A-Z", command=self.sort_alphabetically).grid(row=0, column=3, padx=5, sticky="ew")
        self.btn_filter = ctk.CTkButton(btn_container, text="Masquer complétés", command=self.toggle_filter)
        self.btn_filter.grid(row=0, column=4, padx=5, sticky="ew")

        ctk.CTkLabel(self.toolbar, text="Clic/Shift+Clic sur le fond d'une ligne pour sélectionner. Cliquez sur une coche pour valider.").pack()

        # --- Zone défilante ---
        self.scroll_frame = ctk.CTkScrollableFrame(self)
        self.scroll_frame.grid(row=2, column=0, sticky="nsew") 
        self.scroll_frame.grid_columnconfigure(0, weight=1)
        self.scroll_frame.bind("<Button-1>", lambda e: self.focus_set())

        for i, (line1, line2) in enumerate(self.pairs):
            row_frame = ctk.CTkFrame(self.scroll_frame, fg_color="transparent", corner_radius=0)
            row_frame.grid(row=i, column=0, sticky="ew", pady=1)
            
            # On configure 4 colonnes (Numéro, Coche G, Coche D, Poubelle)
            row_frame.grid_columnconfigure((1, 2), weight=1)
            row_frame.grid_columnconfigure(3, weight=0) 

            lbl_idx = ctk.CTkLabel(row_frame, text=f"#{i+1}", width=40, cursor="hand2")
            lbl_idx.grid(row=0, column=0, padx=10, pady=5)
            lbl_idx.bind("<Button-1>", lambda e, idx=i: self.on_frame_click(e, idx))
            row_frame.bind("<Button-1>", lambda e, idx=i: self.on_frame_click(e, idx))

            cb_l = ctk.CTkCheckBox(row_frame, text=line1.display(), command=lambda idx=i, s=0: self.apply_check(idx, s))
            cb_l.grid(row=0, column=1, padx=10, pady=5, sticky="w")
            
            cb_r = ctk.CTkCheckBox(row_frame, text=line2.display(), command=lambda idx=i, s=1: self.apply_check(idx, s))
            cb_r.grid(row=0, column=2, padx=10, pady=5, sticky="w")

            if self.choices[i] == 0:
                cb_l.select()
            elif self.choices[i] == 1:
                cb_r.select()

            btn_trash = ctk.CTkButton(row_frame, text="🗑", width=40, fg_color="transparent")
            btn_trash.grid(row=0, column=3, padx=5, pady=5)
            btn_trash.bind("<Button-1>", lambda e, idx=i: self.on_trash_click(e, idx))

            self.row_frames.append(row_frame)
            self.checks_l.append(cb_l)
            self.checks_r.append(cb_r)

        # --- Bouton Final ---
        self.btn_done = ctk.CTkButton(self, text="Valider les choix", height=45, 
                                      fg_color="#2ecc71", hover_color="#27ae60",
                                      command=lambda: self.valider(is_auto_save=False))
        self.btn_done.grid(row=3, column=0, pady=10)

    # --- Logique Visuelle (Adaptée pour le mode Clair/Sombre) ---
    def refresh_highlight(self):
        """Met à jour les couleurs de fond avec un gris compatible clair/sombre."""
        for i, frame in enumerate(self.row_frames):
            # On utilise un tuple (Couleur_Mode_Clair, Couleur_Mode_Sombre)
            color = ("gray80", "gray25") if i in self.highlighted_indices else "transparent"
            frame.configure(fg_color=color)

    # --- Les autres méthodes restent identiques ---
    def on_frame_click(self, event, index):
        shift_pressed = (event.state & 0x0001) or (event.state & 1)
        if shift_pressed and self.last_clicked_idx is not None:
            start, end = sorted([self.last_clicked_idx, index])
            self.highlighted_indices = {i for i in range(start, end + 1) if self.row_frames[i].winfo_ismapped()}
        else:
            self.highlighted_indices = {index}
            
        self.last_clicked_idx = index
        self.refresh_highlight()
    
    def apply_check(self, index, side):
        # --- NOUVEAU : Sélection automatique au clic ---
        if index not in self.highlighted_indices:
            self.highlighted_indices = {index}
            self.last_clicked_idx = index
            self.refresh_highlight()


        target_indices = self.highlighted_indices 
        is_checked = self.checks_l[index].get() if side == 0 else self.checks_r[index].get()
        
        for idx in target_indices:
            cl, cr = self.checks_l[idx], self.checks_r[idx]
            if is_checked:
                if side == 0: cl.select(); cr.deselect()
                else: cr.select(); cl.deselect()
            else:
                cl.deselect(); cr.deselect()

    def toggle_filter(self):
        self.hide_completed = not self.hide_completed
        self.btn_filter.configure(text="Rafraîchir / Afficher tout" if self.hide_completed else "Masquer complétés")
        self.refresh_view()

    def refresh_view(self):
        """Affiche ou masque les lignes en combinant le filtre 'complétés' et la logique Regex."""
        pattern_l = self.entry_l.get()
        pattern_r = self.entry_r.get()
        
        try:
            reg_l = re.compile(pattern_l, re.IGNORECASE) if pattern_l else None
            reg_r = re.compile(pattern_r, re.IGNORECASE) if pattern_r else None
        except re.error:
            # Si l'utilisateur est en train de taper une regex incomplète (ex: "[a-"), on attend sans planter.
            return 
            
        for i in range(len(self.pairs)):
            # 1. Vérification des suppressions / complétions
            if i in self.deleted_indices:
                self.row_frames[i].grid_forget()
                continue
                
            is_done = self.checks_l[i].get() or self.checks_r[i].get()
            if self.hide_completed and is_done:
                self.row_frames[i].grid_forget()
                continue
                
            # 2. Vérification Regex
            text_l = self.pairs[i][0].display()
            text_r = self.pairs[i][1].display()
            
            # Cas spécial : Si aucun texte tapé, on affiche tout
            if not pattern_l and not pattern_r:
                show = True
            elif self.logic_mode == "OR":
                # Si OR : l'un ou l'autre doit matcher (si un champ est vide, il vaut False pour ne pas tout afficher)
                m_l = reg_l.search(text_l) if reg_l else False
                m_r = reg_r.search(text_r) if reg_r else False
                show = m_l or m_r
            else: 
                # Si AND ou BIND : les deux doivent matcher (si un champ est vide, il vaut True pour ne pas bloquer l'autre)
                m_l = reg_l.search(text_l) if reg_l else True
                m_r = reg_r.search(text_r) if reg_r else True
                show = m_l and m_r
                
            # 3. Affichage final
            if show:
                self.row_frames[i].grid(row=i, column=0, sticky="ew", pady=1)
            else:
                self.row_frames[i].grid_forget()

    def global_bulk_action(self, side):
        for i in range(len(self.pairs)):
            # CORRECTION : L'action globale n'affecte que les lignes visibles
            if self.row_frames[i].winfo_ismapped():
                cl, cr = self.checks_l[i], self.checks_r[i]
                if side == 0: cl.select(); cr.deselect()
                else: cr.select(); cl.deselect()

    def invert_all_choices(self):
        for i in range(len(self.pairs)):
            # CORRECTION : L'inversion n'affecte que les lignes visibles
            if self.row_frames[i].winfo_ismapped():
                l, r = self.checks_l[i], self.checks_r[i]
                if l.get(): l.deselect(); r.select()
                elif r.get(): r.deselect(); l.select()



    def on_trash_click(self, event, index):
        """Gère le clic sur la poubelle en lisant l'état de la touche Shift."""
        shift_pressed = (event.state & 0x0001) or (event.state & 1)
        
        # 1. Mise à jour de la sélection (Multiple ou Simple)
        if shift_pressed and self.last_clicked_idx is not None:
            start, end = sorted([self.last_clicked_idx, index])
            self.highlighted_indices = {i for i in range(start, end + 1) if self.row_frames[i].winfo_ismapped()}
        else:
            if index not in self.highlighted_indices:
                self.highlighted_indices = {index}
                
        self.last_clicked_idx = index
        self.refresh_highlight()
        
        # 2. Une fois la sélection faite, on lance la suppression globale
        self.delete_pair()


    def delete_pair(self): # Plus besoin de "index"
        """Supprime toutes les paires actuellement en surbrillance."""
        target_indices = self.highlighted_indices 
        if not target_indices:
            return # Sécurité au cas où la sélection est vide
            
        nb_elements = len(target_indices)
        
        # --- Affichage du Pop-up ---
        if nb_elements == 1:
            idx = list(target_indices)[0]
            nom_gauche = self.pairs[idx][0].display()
            nom_droite = self.pairs[idx][1].display()
            msg = f"Voulez-vous vraiment supprimer cette paire ?\n\n• {nom_gauche}\n• {nom_droite}"
        else:
            msg = f"Voulez-vous vraiment supprimer ces {nb_elements} paires ?"
            
        popup = CustomConfirmPopup(self, "Confirmation de suppression", msg)
        self.wait_window(popup)
        
        # --- Action de suppression ---
        if popup.result:
            for idx in list(target_indices):
                self.deleted_indices.add(idx)
                self.row_frames[idx].grid_forget()
                self.checks_l[idx].deselect()
                self.checks_r[idx].deselect()

            # On vide la sélection puisqu'elle a été supprimée
            self.highlighted_indices.clear() 
            self.refresh_highlight()



    def sort_alphabetically(self):
        """Trie les paires selon l'ordre alphabétique de l'élément de gauche."""
        
        # 1. On capture l'état complet de chaque ligne actuelle
        current_state = []
        for i in range(len(self.pairs)):
            current_state.append({
                'pair': self.pairs[i],
                'val_l': self.checks_l[i].get(),
                'val_r': self.checks_r[i].get(),
                'is_deleted': i in self.deleted_indices
            })

        current_state.sort(key=lambda item: item['pair'][0].display().lower())

        self.deleted_indices.clear()
        self.highlighted_indices.clear()
        self.last_clicked_idx = None

        for i, state in enumerate(current_state):
            self.pairs[i] = state['pair']

            # Mise à jour des textes des cases à cocher
            self.checks_l[i].configure(text=self.pairs[i][0].display())
            self.checks_r[i].configure(text=self.pairs[i][1].display())

            # Mise à jour de l'état des coches
            if state['val_l']: self.checks_l[i].select()
            else: self.checks_l[i].deselect()

            if state['val_r']: self.checks_r[i].select()
            else: self.checks_r[i].deselect()

            # Gestion de l'affichage (Corbeille et Filtre)
            if state['is_deleted']:
                self.deleted_indices.add(i)
                self.row_frames[i].grid_forget() # On recache ce qui était supprimé
            else:
                is_done = state['val_l'] or state['val_r']
                if self.hide_completed and is_done:
                    self.row_frames[i].grid_forget() # On respecte le filtre actif
                else:
                    self.row_frames[i].grid(row=i, column=0, sticky="ew", pady=1)
        self.refresh_highlight()

    def cycle_logic(self):
        """Bascule entre les modes OR, AND, et BIND avec code couleur."""
        modes = ["OR", "AND", "BIND"]
        current_idx = modes.index(self.logic_mode)
        self.logic_mode = modes[(current_idx + 1) % 3]
        
        if self.logic_mode == "OR":
            self.btn_logic.configure(text="OR", fg_color="#3498db", hover_color="#2980b9", text_color="white")
            self.entry_r.configure(state="normal")
            
            # --- CORRECTION PLACEHOLDER ---
            if self.entry_r.get() == "":
                self.entry_r.delete(0, "end")
                self.focus_set() # On enlève le focus pour que le texte grisé apparaisse
                
        elif self.logic_mode == "AND":
            self.btn_logic.configure(text="AND", fg_color="#e74c3c", hover_color="#c0392b", text_color="white")
            self.entry_r.configure(state="normal")
            
            # --- CORRECTION PLACEHOLDER ---
            if self.entry_r.get() == "":
                self.entry_r.delete(0, "end")
                self.focus_set()
                
        elif self.logic_mode == "BIND":
            self.btn_logic.configure(text="BIND", fg_color="#f1c40f", hover_color="#f39c12", text_color="black")
            
            self.entry_r.configure(state="normal")
            self.entry_r.delete(0, "end")
            self.entry_r.insert(0, self.entry_l.get())
            self.entry_r.configure(state="disabled")
            
        self.refresh_view()

    def on_regex_change(self, event=None):
        """Si on tape du texte et qu'on est en BIND, on réplique à droite."""
        if self.logic_mode == "BIND":
            self.entry_r.configure(state="normal")
            self.entry_r.delete(0, "end")
            self.entry_r.insert(0, self.entry_l.get())
            self.entry_r.configure(state="disabled")
            
        self.refresh_view()

    def valider(self, is_auto_save=False):
        choix_valides = []
        paires_restantes = []
        paires_supprimees = []
        
        for i in range(len(self.pairs)):
            if i in self.deleted_indices:
                paires_supprimees.append(self.pairs[i])
            else:
                # On regarde l'état actuel des cases
                choice = None
                if self.checks_l[i].get(): choice = 0
                elif self.checks_r[i].get(): choice = 1
                
                # Validation explicite : on enregistre le résultat
                if not is_auto_save and choice is not None:
                    choix_valides.append(self.pairs[i][choice])
                else:
                    # Sauvegarde auto (ou rien n'est coché) : on renvoie la paire ET sa mémoire !
                    paires_restantes.append((self.pairs[i], choice))
                    
        self.on_complete(choix_valides, paires_restantes, paires_supprimees, is_auto_save)
            
