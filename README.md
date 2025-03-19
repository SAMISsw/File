# File

```swift
import SwiftUI
import UniformTypeIdentifiers
import QuickLook
import MobileCoreServices
import AVFoundation
import PDFKit
import SyntaxEditor

struct FileManagerView: View {
    @State private var files: [URL] = []
    @State private var selectedFolder: URL?
    @State private var showQuickLook = false
    @State private var previewFile: URL?
    @State private var searchQuery: String = ""
    @State private var editingFileURL: URL?
    @State private var fileContent: String = ""

    private let fileManager = FileManager.default
    private let directory: URL = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask).first!

    var body: some View {
        NavigationStack {
            VStack {
                TextField("Buscar arquivos...", text: $searchQuery)
                    .padding()
                    .background(Color(.systemGray6))
                    .cornerRadius(8)
                    .padding(.horizontal)

                List {
                    ForEach(filteredFiles(), id: \.self) { file in
                        HStack {
                            Image(systemName: file.hasDirectoryPath ? "folder.fill" : "doc.fill")
                                .foregroundColor(file.hasDirectoryPath ? .blue : .gray)
                                .font(.title)

                            VStack(alignment: .leading) {
                                Text(file.lastPathComponent)
                                Text(fileSizeAndDate(file))
                                    .font(.caption)
                                    .foregroundColor(.gray)
                            }

                            Spacer()
                            Menu {
                                Button(action: { openFile(file) }) { Image(systemName: "eye") }
                                Button(action: { editTextFile(file) }) { Image(systemName: "pencil") }
                                Button(action: { copyFile(file) }) { Image(systemName: "doc.on.doc") }
                                Button(action: { moveFile(file) }) { Image(systemName: "folder.badge.plus") }
                                Button(action: { shareFile(file) }) { Image(systemName: "square.and.arrow.up") }
                                Button(action: { renameFile(file) }) { Image(systemName: "pencil.and.ellipsis.rectangle") }
                                Button(action: { deleteFile(at: file) }) { Image(systemName: "trash") }
                            } label: {
                                Image(systemName: "ellipsis.circle")
                            }
                        }
                        .onTapGesture {
                            if file.hasDirectoryPath {
                                selectedFolder = file
                                loadFiles(from: file)
                            }
                        }
                    }
                }
            }
            .navigationTitle(selectedFolder?.lastPathComponent ?? "Arquivos")
            .onAppear { loadFiles() }
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button(action: { createFolder() }) { Image(systemName: "folder.badge.plus") }
                }
            }
            .sheet(isPresented: $showQuickLook) {
                QuickLookPreview(fileURL: previewFile!)
            }
            .sheet(item: $editingFileURL) { fileURL in
                CodeEditorView(fileURL: fileURL, fileContent: $fileContent)
            }
        }
    }

    func filteredFiles() -> [URL] {
        guard !searchQuery.isEmpty else { return files }
        return files.filter { $0.lastPathComponent.lowercased().contains(searchQuery.lowercased()) }
    }

    func loadFiles(from folder: URL? = nil) {
        let path = folder ?? directory
        do {
            files = try fileManager.contentsOfDirectory(at: path, includingPropertiesForKeys: [.contentModificationDateKey, .fileSizeKey])
        } catch {
            print("Erro ao carregar arquivos: \(error)")
        }
    }

    func openFile(_ file: URL) {
        previewFile = file
        showQuickLook = true
    }

    func editTextFile(_ file: URL) {
        if let text = try? String(contentsOf: file, encoding: .utf8) {
            fileContent = text
            editingFileURL = file
        }
    }

    func copyFile(_ file: URL) {
        let newURL = file.deletingLastPathComponent().appendingPathComponent("Cópia - \(file.lastPathComponent)")
        do {
            try fileManager.copyItem(at: file, to: newURL)
            loadFiles()
        } catch {
            print("Erro ao copiar arquivo: \(error)")
        }
    }

    func moveFile(_ file: URL) {
        let newURL = directory.appendingPathComponent("Movido - \(file.lastPathComponent)")
        do {
            try fileManager.moveItem(at: file, to: newURL)
            loadFiles()
        } catch {
            print("Erro ao mover arquivo: \(error)")
        }
    }

    func shareFile(_ file: URL) {
        let activityVC = UIActivityViewController(activityItems: [file], applicationActivities: nil)
        if let windowScene = UIApplication.shared.connectedScenes.first as? UIWindowScene {
            windowScene.windows.first?.rootViewController?.present(activityVC, animated: true, completion: nil)
        }
    }

    func renameFile(_ file: URL) {
        let newURL = file.deletingLastPathComponent().appendingPathComponent("Renomeado - \(file.lastPathComponent)")
        do {
            try fileManager.moveItem(at: file, to: newURL)
            loadFiles()
        } catch {
            print("Erro ao renomear arquivo: \(error)")
        }
    }

    func deleteFile(at url: URL) {
        do {
            try fileManager.removeItem(at: url)
            loadFiles()
        } catch {
            print("Erro ao excluir arquivo: \(error)")
        }
    }

    func createFolder() {
        let folderURL = directory.appendingPathComponent("Nova Pasta")
        do {
            try fileManager.createDirectory(at: folderURL, withIntermediateDirectories: true, attributes: nil)
            loadFiles()
        } catch {
            print("Erro ao criar pasta: \(error)")
        }
    }

    func fileSizeAndDate(_ file: URL) -> String {
        do {
            let attributes = try file.resourceValues(forKeys: [.fileSizeKey, .contentModificationDateKey])
            let fileSize = attributes.fileSize ?? 0
            let date = attributes.contentModificationDate ?? Date()
            let formattedDate = DateFormatter.localizedString(from: date, dateStyle: .short, timeStyle: .short)
            return "\(fileSize / 1024) KB - \(formattedDate)"
        } catch {
            return "Informações indisponíveis"
        }
    }
}

struct CodeEditorView: View {
    @Binding var fileContent: String
    var fileURL: URL

    @State private var syntaxHighlighter = SyntaxHighlighter()

    var body: some View {
        VStack {
            TextEditor(text: $fileContent)
                .padding()
                .background(Color(.systemGray6))
                .cornerRadius(8)
                .font(.custom("Courier New", size: 14))

            HStack {
                Button(action: {
                    saveFile()
                }) {
                    Image(systemName: "square.and.arrow.down")
                    Text("Salvar")
                }
                .padding()
                
                Button(action: {
                    closeEditor()
                }) {
                    Image(systemName: "xmark.circle.fill")
                    Text("Fechar")
                }
                .padding()
            }
        }
        .navigationTitle(fileURL.lastPathComponent)
        .onAppear {
            loadFileContent()
        }
    }

    func loadFileContent() {
        if let content = try? String(contentsOf: fileURL, encoding: .utf8) {
            fileContent = content
        }
    }

    func saveFile() {
        do {
            try fileContent.write(to: fileURL, atomically: true, encoding: .utf8)
        } catch {
            print("Erro ao salvar arquivo: \(error)")
        }
    }

    func closeEditor() {
        fileContent = ""
    }
}

struct QuickLookPreview: UIViewControllerRepresentable {
    var fileURL: URL

    func makeUIViewController(context: Context) -> QLPreviewController {
        let controller = QLPreviewController()
        controller.dataSource = context.coordinator
        return controller
    }

    func updateUIViewController(_ uiViewController: QLPreviewController, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, QLPreviewControllerDataSource {
        let parent: QuickLookPreview

        init(_ parent: QuickLookPreview) {
            self.parent = parent
        }

        func numberOfPreviewItems(in controller: QLPreviewController) -> Int {
            return 1
        }

        func previewController(_ controller: QLPreviewController, previewItemAt index: Int) -> QLPreviewItem {
            return parent.fileURL as QLPreviewItem
        }
    }
}
```
